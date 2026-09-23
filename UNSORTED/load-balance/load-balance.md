# sticky sessions (session persistence)

When a user logs into an online shopping website, the load balancer continues sending all requests from that user to the same server. This allows the user's login session and shopping cart to remain available without requiring re-authentication.

# SSL/TLS termination

the process in which a load balancer decrypts incoming HTTPS traffic before forwarding the request to backend servers. This reduces the processing overhead on application servers and simplifies SSL certificate management.

Example: A banking website receives encrypted HTTPS requests from users. The load balancer decrypts the requests, forwards them to the web servers over the internal network, and then encrypts the responses before sending them back to users.

# where to add load balancer
