# consul
The heart of Consul is a particular class of distributed datastore with properties that make it ideal for cluster configuration and coordination. Some call them lock servers, but I call them "config stores" since it more accurately reflects their key-value abstraction and common use for shared configuration.

These specialized datastores are defined by their use of a consensus algorithm requiring a quorum for writes and generally exposing a simple key-value store. This key-value store is highly available, fault-tolerant, and maintains strong consistency guarantees. This can be contrasted with a number of alternative clustering approaches like master-slave or two-phase commit, all with their own benefits, drawbacks, and nuances.

Since Zookeeper came out, the subsequent config stores have been trying to simplify. Both in terms of user interface, ease of operation, and implementation of the consensus algorithms. However, they're all based on this very expressive, but lowest common denominator abstraction of a key-value store.

Consul is the first to build on top of this abstraction by also providing specific APIs around the semantics of common config store functions, namely service discovery and locking. It also does it in a way that's very thoughtful about those particular domains.

For example, a directory of services without service health is actually not a very useful one. This is why Consul also provides monitoring capabilities. Consul monitoring is comparable, and even compatible, with Nagios health checks. What's more, Consul's agent model makes it more scalable than centralized monitoring systems like Nagios.
 
 Ver http://progrium.com/images/content/embassy/consul_layers.png
 
 A good way to think of Consul is broken into 3 layers. The middle layer is the actual config store, which is not that different from etcd or Zookeeper. The layers above and below are pretty unique to Consul.

Before Consul, HashiCorp developed a host node coordinator called Serf. It uses an efficient gossip protocol to connect a set of hosts into a cluster. The cluster is aware of its members and shares an event bus. This is primarily used to know when hosts come and go from the cluster, such as during a host failure. But in Serf the event bus was also exposed for custom events to trigger user actions on the hosts.

Consul leverages Serf as a foundational layer to help maintain its cluster. For the most part, it's more of an implementation detail. However, I believe in an upcoming version of Consul, the Serf event bus will also be exposed in the Consul API.

The key-value store in Consul is very similar to etcd. It shares the same semantics and basic HTTP API, but differs in subtle ways. For example, the API for reading values lets you optionally pick a consistency mode. This is great not just because it gives users a choice, but it documents the realities of different consistency levels. This transparency educates the user about the nuances of Consul's replication model.

On top of the key-value store are some other great features and APIs, including locks and leader election, which are pretty standard for what people originally called lock servers. Consul is also datacenter aware, so if you're running multiple clusters, it will let you federate clusters. Nothing complicated, but it's great to have built-in since spanning multiple datacenters is very common today.

However, the killer feature of Consul is its service catalog. Instead of using the key-value store to arbitrarily model your service directory as you would with etcd or Zookeeper, Consul exposes a specific API for managing services. Explicitly modeling services allows it to provide more value in two main ways: monitoring and DNS.

Monitoring is normally discussed independent of service discovery, but it turns out to be highly related. Over the years, we've gotten better at understanding the importance of monitoring service health in relation to service discovery.

With Zookeeper, a common pattern for service presence, or liveness, was to have the service register an "ephemeral node" value announcing its address. As an ephemeral node, the value would exist as long as the service's TCP session with Zookeeper remained active. This seemed like a rather elegant solution to service presence. If the service died, the connection would be lost and the service listing would be dropped.

In the development of doozerd, the authors avoided this functionality, both for the sake of simplicity and that they believed it encouraged bad practice. The problem with relying on a TCP connection for service health is that it doesn't exactly mean the service is healthy. For example, if the TCP connection was going through a transparent proxy that accidentally kept the connection alive, the service could die and the ephemeral node may continue to exist.


 Instead, they implemented values with an optional TTL. This allowed for the pattern of actively updating the value if the service was healthy. TTL semantics are also used in etcd, allowing the same active heartbeat pattern. Consul supports TTL as well, but primarily focuses on more robust liveness mechanisms. In the discovery layer I helped design for Flynn, our client library lets you register your service and it will automatically heartbeat for you behind the scenes.
 
 This is generally effective for service presence, but it might not take the lesson to heart. Blake Mizerany, the co-author of doozerd and now maintainer of etcd, will stress the importance of meaningful liveness checks. In other words, there is no one-size-fits-all. Every service performs a different function and without testing that specific functionality, we don't actually know that it's working properly. Generic heartbeats can let us know if the process is running, but not that it's behaving correctly enough to safely accept connections.

Specialized health checks are exactly what monitoring systems give us, and Consul gives us a distributed monitoring system. Then it lets us choose if we want to want to associate a check with a service, while also supporting the simpler TTL heartbeat model as an alternative. Either way, if a service is detected as not healthy, it's hidden from queries for active services.

Consul is much less popular in the Docker world. Perhaps this is just due to less of a focus on containers at HashiCorp, which is contrasted by the heavily container-oriented mindset of the etcd maintainers at CoreOS.

