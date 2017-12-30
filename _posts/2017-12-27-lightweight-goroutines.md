---
layout: post
title:  "The Magic Behind Go's Lightweight Goroutines"
---
One feature that is often brought up when discussing the Go programming language is the use of goroutines; a type of lightweight processes that makes it possible to run thousands of goroutines concurrently. Many other languages use threading provided by the operating systems to support concurrent tasks. The downside with threads is that they are relatively heavyweight, and can therefore only run hundreds of threads before experiencing scalability issues. These problems are especially evident in the era of real time updates with a lot of client connections.

Often, we're satisfied with saying that goroutines are a more “lightweight” version of threads. But how can they be more lightweight, while still having the same functionality. I ended up diving into the source code of the Go runtime to find the answer. In this article, I‘ll show you how Go’s scheduling mechanism is working by implementing a simple Go program that handles the routines itself. That way, we can get around the low-level parts, while keeping it simple.

# Scheduling Tasks

Goroutines are built on the principle of event-driven architecture. When an event happens, a task related to the event is put on a queue. The event loop goes through the queue and executes the tasks one by one. What happens if the task triggered takes a long time to execute? Then all other events on the queue get stalled. Isn't that the reason we want to use threads, so we can assure responsiveness? If some task claims the processor for too long, the thread gets interrupted by the scheduler, that in turn lets other threads do their tasks. The problem is that we get a lower throughput, as we have to spend time putting away half-done work while switching between tasks. To be more concrete, half-done work can for example be variables that we want to multiply together and is occupying a couple of CPU registers. In that case, we would have to swap all those registers back and forth.

Goroutines tries to solve the stalling problem of the event-driven approach by letting the task invoke the scheduler itself when appropriate. This typically occurs when the task has to wait for some input or output and doesn’t have anything to do. In Go 1.2 the scheduler is also [invoked when doing function calls](https://golang.org/doc/go1.2#preemption), as the CPU registers has to be given over to the calle anyways. Go also reduces the risk of stalling by running parallel event loops on different CPU cores, but we'll not bring that up here.

# An Echoserver Example

Let’s start with a simple server that starts a goroutine for each new TCP connection. Error handling is omitted here for brevity, but you can find the full code on [my Github repo](https://github.com/HadarGreinsmark/goroutines-poll-example).

```go
func main() {
    addr, _ := net.ResolveTCPAddr("tcp", ":7777")
    listener, _ := net.ListenTCP("tcp", addr)
    replyGoroutine(listener)
}

func replyGoroutine(listener net.Listener) {
    for {
        conn, _ := listener.Accept()

        go func() {
            buf := make([]byte, 16)
            conn.Read(buf)
            log.Printf("received: %s", buf)
            conn.Write(bytes.ToUpper(buf))
            conn.Close()
        }()
    }
}
```
All the `Accept()`, `Read()` and `Write()` calls wait for some external operation to finish, which would be the perfect time to switch to another task while waiting. The Go environment invoke the scheduler `runtime.Gosched()` on these waiting points, so the process can switch to another goroutine that has work to do.

We can test this code in Bash by using Netcat:


```bash
$ echo "Hello World" | nc localhost 7777
HELLO WORLD
```

There are other programming languages where you can achieve this manually. Node.js based on JavaScript follows similar practices as Go, though the developer has to to do this by hand when writing the code. Here is an example of the same code in Node.js:

```javascript
var net = require('net');

var server = net.createServer(function(socket) {
    socket.on('data', function(data) {
        var buf = data.toString();
        console.log("received:", buf)
        socket.write(buf.toUpperCase(), function() {
            socket.end()
        });
    });
})

server.listen(7777);
```

As you can see, for each call where we have to wait for an external operation, we give a callback function as argument. `createServer()` for accepting new connections, `on()` to read data and `write()` for filling the buffer when having space available. All of those have to wait until an external operation is finished. Meanwhile, the event loop can perform other things while waiting. As you can see, the event driven approach can sometimes be hard to read as you have to define nested function definitions and keep track of variable scopes. Newer versions of JavaScript try to deal with this issue using async/await. But Go manages all of these things for you, so you don't have to think about it at all.

# Go Without Goroutines

Above, we have been satisfied with the explanation that we do other things while we "wait for some external operation to finish". But how does that really work? To fully understand that, we have to dive into how UNIX polling and file descriptors work. We do it by using them in the same example above, and implementing the scheduling logic of our own pseudo-goroutines ourselves.

A file descriptor is a resource that can handle input, output and other related operations on external resources. They're used when reading/writing a file, but also for listening on a TCP port for new clients and handle an open TCP connection. We can access these resources by using functions like `accept()`, `read()` and `write()` in UNIX. The problem is that these functions can only handle one resource at a time. Fortunately, we can use UNIX polling to watch for events on several resources at the same time. 

In our example, we’re using the [`epoll`](http://man7.org/linux/man-pages/man7/epoll.7.html) system call for this, that is only supported on Linux. The Go runtime uses the same system call in a similar fashion. In Go, you can access system calls through the `golang.org/x/sys/unix` package. It gives you a Go friendly syscall-API to work with.

```go
type GoroutineState struct {
	connFile *os.File
	buffer   []byte
}
```

The variables related to each goroutine is emulated using our `GoroutineState`, that is stored in a map with the TCP connection's file descriptor as key.

Next, we implement the event loop with `EpollWait()` that here watches for file descriptor events from the TCP listener and the TCP connections. `EpollCtl()` is used for changing the set of resources to watch for events. Check out the full code at [this Github repo](https://github.com/HadarGreinsmark/goroutines-poll-example) for full error handling.


```go
func replierPoll(listener *net.TCPListener) {
    epollFd, _ := unix.EpollCreate(8)

    // UNIX represents a TCP listener socket as a file
    listenerFile, _ := listener.File()

    // Add the TCP listener to the set of file descriptors being polled
    listenerPoll := unix.EpollEvent{
        Fd:     int32(listenerFile.Fd()),
        Events: unix.POLLIN, // POLLIN triggers on accept()
        Pad:    0,           // Arbitary data
    }
    unix.EpollCtl(epollFd, unix.EPOLL_CTL_ADD, int(listenerPoll.Fd), &listenerPoll)

    // Map EpollEvent.Pad to the connection state
    states := map[int]*GoroutineState{}

    for {
        // Wait infinitely until at least one new event is happening
        var eventsBuf [10]unix.EpollEvent
        unix.EpollWait(epollFd, eventsBuf[:], -1)

        // Go though every event occured; most often len(eventsBuf) == 1
        for _, event := range eventsBuf {
            if event.Fd == listenerPoll.Fd {
                // Handle new connection
                // AcceptTCP() will now return immediately
                conn, _ := listener.AcceptTCP()

                // Equal to creating a new goroutine
                newState := addNewClientPoll(epollFd, conn)
                fd := int(newState.connFile.Fd())
                states[fd] = newState
                continue
            }

            // Handle existing connection
            fd := int(event.Pad)
            state := states[fd]

            if event.Events == unix.POLLIN {
                state.connFile.Read(state.buffer)
                log.Printf("received: %s", state.buffer)

                // Equal to switching away the goroutine
                newPoll := event
                newPoll.Events = unix.POLLOUT
                unix.EpollCtl(epollFd, unix.EPOLL_CTL_MOD, fd, &newPoll)

            } else if event.Events == unix.POLLOUT {
                state.connFile.Write(bytes.ToUpper(state.buffer))
                state.connFile.Close()

                // Equal to stopping the goroutine
                unix.EpollCtl(epollFd, unix.EPOLL_CTL_DEL, fd, nil)
                delete(states, fd)
            }
        }
    }
}
```

Our event loop is waiting for three types of events:

* New connection: TCP port listener triggers `POLLIN` event, `AcceptTCP()` is promised to return immediately
* Receive data from TCP client: client socket triggers `POLLIN` event, `Read()` is promised to return immediately
* Space available in buffer to be sent to TCP client: client socket triggers `POLLOUT` event, `Write()` is promised to return immediately

Here is the code for adding the new pseudo-goroutine to a new connection:

```go
func addNewClientPoll(epollFd int, conn *net.TCPConn) *GoroutineState {
    connFile, _ := conn.File()
    conn.Close() // Close this an use the connFile copy instead

    newState := GoroutineState{
        connFile: connFile,
        buffer:   make([]byte, 16),
    }
    fd := int(connFile.Fd())

    connPoll := unix.EpollEvent{
        Fd:     int32(fd),
        Events: unix.POLLIN, // POLLIN triggers on accept()
        Pad:    int32(fd),   // So we can find states[fd] when triggered
    }
    unix.EpollCtl(epollFd, unix.EPOLL_CTL_ADD, fd, &connPoll)

    return &newState
}
```

`epoll` only works on Linux. But similar system calls can be found on other operating systems, like `kqueue()` on Mac/BSD and the less scalable `poll()` on POSIX systems.

# Conclusion
As we can see, Go uses the techniques of event-driven architecture without the programmer having to know it. JavaScript/Node.js follows similar practices, but requires the programmer to write code for it and think through potential stalling problems. In Go, you rarely have to think about it. From our example, we can also see that it's very easy to access UNIX system calls as the unix package offers a Go-friendly wrapper that can be used for low-level programming.

# Further Reading
* How to stop Go's scheduler loop from working: [*A pitfall of golang scheduler*](http://www.sarathlakshman.com/2016/06/15/pitfall-of-golang-scheduler)
* Deeper coverage of the scheduler concepts: [*Analysis of the Go runtime scheduler*](http://www.cs.columbia.edu/~aho/cs6998/reports/12-12-11_DeshpandeSponslerWeiss_GO.pdf)
