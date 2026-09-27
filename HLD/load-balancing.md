Load Balancing is a key concept in system design. One simple way would be hashing all requests and then sending them to the assigned server.

The standard way to hash objects is to map them to a search space and then transfer the load to the mapped computer. A system using this policy is likely to suffer when new nodes are added or removed from it. 

One term you would hear in system design interviews is Fault Tolerance, in which case a machine crashes. And Scalability, in which devices must be added to process more requests. Another term used often is request allocation. This means assigning a request to a server.

Load balancing is often tied with service discovery and global locks. The type of load we want to balance is that of sticky sessions.

Notes to self:
* Load balancing distributes requests across servers.
* You can use `hash(r_id) % n_servers` to get the server index for a request `r_id`.
    -> Drawback: if you add an extra server `n_servers` changes and `r_id` will end up on a different server. This is bad because often we want to map requests with the same ids consistently to the same servers (there could e.g. be cached data there that we want to reuse).
* "Consistent hashing" hashes with a constant denominator `M`, e.g. `hash(r_id) % M`, and then maps the resulting integer onto a server index. Each server has a range of integers that map to their index.
* The pie example demonstrates, that if an extra server is added, the hashing function stays the same, and one can then change the range-to-server-index mapping slightly so that it remains likely that an `r_id` gets mapped to the same server as before the server addition.