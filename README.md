# GoLoadBalancer
A Simple Load Balancer with Go

Load Balancers play a key role in web architecture by distributing the load to a group of backend servers. This makes services more scalable and more reliable.


## Useful Links:
- Professional load balancer: [NGINX](https://www.nginx.com/)
- [What Is Load Balancing?](https://www.nginx.com/resources/glossary/load-balancing/)


## How does our Simple Load Balancer work?
Load Balancers have different strategies for distributing the load across a set of backends.


## Load Balancing Techniques:
Round Robin:
* Distribute load equally, assume all backends have the same processing power

Weighted Round Robin:
* Additional weights can be given considering the backend's processing power

Least Connections:
* Load is distributed to the server with least active connections

## Our Simple Load Balancer: Round Robin
Our Simple Load Balancer implements using the simplest among these methods: **Round Robin**

![Round Robin](./img/round_robin.png)

## Round Robin Selection
`Round Robin` technique gives equal opportunities for workers to perform tasks in turns.

![Round Robin Selection on incoming requests](./img/round_robin_selection.png)

This happens cyclically, but we can't directly use that...

## **What if a backend is down?**
We don't want to router traffic to dead server.

So this cannot be used directly unless we put some conditions on it:
- We need to **route traffic only to backends which are up and running.**

