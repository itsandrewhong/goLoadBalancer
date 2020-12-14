# GoLoadBalancer
A Simple Load Balancer with Go.
It uses Round Robin algorithm to send requests into set of backends.


# Intro
Load Balancers play a key role in web architecture by distributing the load to a group of backend servers. This makes services more scalable and more reliable.

# How to use
```bash
Usage:
  -backends string
        Load balanced backends, use commas to separate
  -port int
        Port to serve (default 3030)
```

## Example:

To add followings as load balanced backends:
- http://localhost:3031
- http://localhost:3032
- http://localhost:3033
- http://localhost:3034

```bash
simple-lb.exe --backends=http://localhost:3031,http://localhost:3032,http://localhost:3033,http://localhost:3034
```

# Useful Links:
- Professional load balancer: [NGINX](https://www.nginx.com/)
- [What Is Load Balancing?](https://www.nginx.com/resources/glossary/load-balancing/)


# Topics Covered:
* Round Robin Selection
* ReverseProxy from the standard library
* Mutexes
* Atomic Operations
* Closures
* Callbacks
* Select Operation


# Can we do Better?:
* Use a heap for sort out alive backends to reduce search surface
* Collect statistics
* Implement weighted round-robin / least connections
* Add support for a configuration file


# How does our Simple Load Balancer work?
Load Balancers have different strategies for distributing the load across a set of backends.


# Load Balancing Techniques:
## Round Robin:
Distribute load equally, assume all backends have the same processing power

## Weighted Round Robin:
* Additional weights can be given considering the backend's processing power

## Least Connections:
* Load is distributed to the server with least active connections


# Our Simple Load Balancer: Round Robin
Our Simple Load Balancer implements using the simplest among these methods: **Round Robin.**
![Round Robin](./img/round_robin.png)


# Round Robin Selection
`Round Robin` technique gives equal opportunities for workers to perform tasks in turns.
![Round Robin Selection on incoming requests](./img/round_robin_selection.png)


## **What if a backend is down?**
We don't want to router traffic to dead server. So this cannot be used directly unless we put some conditions on it:
- We need to **route traffic only to backends which are up and running.**


# Design Process
## 1. Reverse Proxy
HTTP Handler that takes an incoming request and sends it to another server, proxying the response back to the client. [Read More about Go's Reverse Proxy util](https://golang.org/pkg/net/http/httputil/#ReverseProxy)

Initiate the `ReverseProxy` with the associated `URL` in the `Backend` so that `ReverseProxy` will route our requests to the `URL`.


## 2. Selection Process
We need to skip dead backends during the next pick. Multiple clients will connect to the load balancer and when each of them requests a next peer to pass the traffic on race conditions could occur.

To prevent it, lock the `ServerPool` with `mutex`. 

But we just want to incrase the counter by one. Ideal solution: Make increment atomically. `atomic`


## 3. Picking up an Alive Backend
Requests are routed in a cycle for each backend. `GetNext()` ->  Returns a value 0 - length of the slice

Traverse from next to the entire list, and cap it between slice length by using `mod` operator.


## 4. Avoid Race Conditions in Backend struct
`Backend` structure has a variable which could be modified or accessed by different goroutines same time. Use `RWMutex` to serialize the access to the `Alive`


## 5. Route traffic only to healthy backends
In order to check if backend is healthy or not, we have to try out a backend and check whether it is alive.

Two ways to do this:
1. **Active**: While performing the current request, we find the selected backend is unresponsive, mark it down.
2. **Passive**: Ping backends on fixed intervals and check status.


## 7. Actively Checking for Healthy Backends
`ReverseProxy` triggers a callback function, `ErrorHandler` on any error.

Due to temporary errors the server may reject the requests and it may be available after a short delay (possibly the server ran out of sockets to accept more clients)

So, put a timer to delay the retry for around 10 milliseconds, at most of 3 tries. And increment the retry count with every request. After every retry failed, mark this backend as down.

Next, attempt a new backend to the same request by keeping a count of attempts using the context package. After increasing the attempt count, pass it back to `lb` to pick a new peer to process the request. Since we can't do this indefintely, we need to check from `lb` whether the maximum attempts already taken before processing the request further.


## 8. Use of context
`context` package allows to store useful data in an Http request. Use this to track request specific data such as Attempt count and Retry count.

First, specify keys for the context. It is recommended to use non-colliding integer keys rather than strings.

Go provides `iota` keyword to implement constants incrementally, each containing a unique value. Then, we retrieve the value as usually we do with a HashMap like follows.


## 9. Passive Health Checks
Passive Health Checks allow to recover dead backends or identify them. Ping the backends with fixed intervals to check their status.

To ping, we try to establish a TCP connection. If the backend responses, we mark it as alive. This method can be changed to call a specific endpoint like `/status`.

Make sure to close the connection once it established to reduce the additional load in the server. Otherwise, it will try to maintain the connection and it would run out of resources eventually.

