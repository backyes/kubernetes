# Kubernetes Networking

Kubernetes is a distributed operating system that schedules containers
to run on a cluster.  At the core of Kubernetes, it is the networking
mechanism.  We have to understand this before we can deploy and use
Kubernetes.  To understand Kubernetes networking, we need to start
from Docker networking.

## Docker Networking

Docker networking is described
[here](http://kubernetes.io/docs/admin/networking/#docker-model),
which can be illustrated with the following figure:

<img src="./docker.png" width=800 />

- On each node, Docker creates a virtual bridge named `docker0`, which
  is responsible for allocating local IP addresses for containers.
  This is called *container IP*.
- Processes running within a container share the same container IP,
  but with different in-container ports.  So when `process 1` connects
  to `process 2`, it connects to in-container address
  `localhost:2379`.
- Processes running in different containers but on the same node can
  identify each other with container IP.  For example, when `process
  4` connects to `process 2`, it uses `container-IP-1:2379`.
- The communication between processes on different nodes is
  complicated.  For `process 3` to talk to `processs 7`, it connects
  to `host-IP-2:8080`, where `8080` is a host port mapped to a port in
  a certain container.

A primary difficulty with the Docker networking model is that in order
to expose a process in a container, we need to map its port to a host
port -- but to which port?
