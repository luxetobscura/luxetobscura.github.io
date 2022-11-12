---
layout: post
title: ":cookie: Load Balancers and HTTP Cookie Persistence"
date: 2022-06-05
---
When a client connects to a load-balanced virtual server to initiate an HTTP session, the load balancer can insert a platform-specific cookie in the HTTP response headers that typically contains the hashed/encoded value of a specific node `address:port` combination. The client's browser stores the persistence cookie and sends it on each subsequent request, and the load balancer directs the client's connection to the node specified in the cookie.

## Prerequisites

- If a web application is secured with HTTPS, it should go without saying that any operation requiring the load balancer to modify the HTTP request or response (such as inserting an HTTP cookie header) will require the load balancer to terminate the client's TLS connection. That means configuring a public/private key pair for the X.509 certificate that the load balancer will present to the client, and sometimes also means ensuring that the load balancer can communicate with the backend web servers over HTTPS, which may require adding CA certificates to the load balancer if the backend servers use a private CA.
- There may be other platform-specific pitfalls. For example, F5 BIG-IPs make load balancing decisions at layer 4 by default, and may continue to send a client to the same node with which the client initially established its TCP connection, even if the client's persistence cookie value changes. Additional configuration is required in order for the BIG-IP to (a) gain visibility into the client's HTTP session, and (b) ensure that the client's persistence cookie is evaluated on each HTTP request. F5 article [K7964](https://support.f5.com/csp/article/K7964) has details about these intricacies.

## Troubleshooting

When removing/decommissioning nodes from a load-balanced pool fronted by a virtual server that relies on cookie-insert persistence, keep in mind that clients will retain the persistence cookie that tells the load balancer to send them to a specific node. If not managed correctly, this will tell the load balancer to send clients to one of the nodes that was removed from the pool, which will usually return a TCP reset to the client since the node is no longer part of the load balancer's connection table.

Story time: a coworker of mine recently observed this behavior in the form of user complaints when they decommissioned two web servers, and proposed a few solutions:

1. Tell users to clear their cookie for the website. Far from ideal - getting users to do what you want them to is often harder than getting computers to do what you want them to. However, the message would be simple enough for a helpdesk technician to communicate to the average user as a workaround in a pinch.
2. Change the persistence configuration on the virtual server. The virtual server in question was configured to present HTTP session persistence cookies by default. This meant that the persistence cookie lifespan was at the mercy of each client's browser to determine what an "HTTP session" was. For most modern browsers, this almost certainly meant "until the browser window is closed." While a time-based cookie expiration could be configured, the existing cookies cached by client browsers still had to be addressed.

In light of those challenges, I proposed the following instead:

- Add the nodes back to the pool in a disabled state. This would allow the load balancer to make a more informed load balancing decision based on the state of the pool members. Depending on the load balancing platform and the pool/node configuration options available, the response could vary from ungraceful (TCP/HTTP error, followed by new load balancing decision and subsequent persistence cookie on page refresh) to seamless (detect that the target node is down, send the client to a healthy node, and return a new persistence cookie).
- Add logging to the virtual server to inspect the target specified in each persistence cookie, and filter by the addresses of the servers to be decommissioned. Then, remove the servers from the pool for good once a week had passed or the observed logs reported zero hits within a 48-hour window, whichever of those two events came first.

The issue ended up being moot because a short conversation with the developers of the web application revealed that the application was stateless, obviating the need for persistence in the first place. I'll spare you the didactic conclusion that is common with such tales, but I hope this was helpful anyways. :sweat_smile:
