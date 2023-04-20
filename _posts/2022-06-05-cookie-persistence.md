---
layout: post
title: ":cookie: Load Balancers and HTTP Cookie Persistence"
date: 2022-06-05
updated: 2023-04-19
---
What does a cookie have to do with persistence?

Before we begin: There are many excellent articles about the theory and execution of **session persistence**, also known as "sticky sessions;" if the term is unfamiliar, research will be left as an exercise to the reader. My aim in this article is to outline some of the knowledge and pitfalls surrounding cookie-based session persistence that I've learned about by working with stateful web apps in the wild.

## What problem does cookie persistence solve?

Back when stateful web apps were the norm, several methods for session persistence emerged. Arguably the best-known method uses information about a client's network to route requests. However, a client's source network wasn't guaranteed to provide sufficient identifying information about the actual user, as their source address could be a NAT.

Enter cookie-based persistence. Why have the load balancer keep track of each user's session when you can have the user tell you themselves?

## How does it work?

The exact method varies by platform, but the general idea is that the load balancer returns a cookie that tells the client's browser which backend server has their session data. The load balancer maintains knowledge of the cookie values either by storing them in memory (Citrix ADC, NGINX), or determining them algorithmically (F5 BIG-IP, haproxy).

1. A client initiates a new HTTP session to the load balancer.
1. The load balancer routes the client's request to a backend server based on the configured load balancing algorithm, then records the backend server that was selected.
1. The load balancer inserts a cookie value into the HTTP response headers containing some information about the backend server.
1. On subsequent requests, the client's browser sends the cookie, and the load balancer routes the client's request to the backend server identified by the cookie value.

## Pitfalls

1. This should go without saying, but if the load balancer is serving connections over HTTPS, it must also terminate TLS to insert cookies into the HTTP response.
1. There may be other platform-specific pitfalls. For example, F5 BIG-IPs make load balancing decisions at layer 4 by default, and may continue to send a client to the same node with which the client initially established its TCP connection, even if the client's persistence cookie value changes. Additional configuration is required in order for the BIG-IP to (a) gain visibility into the client's HTTP session, and (b) ensure that the client's persistence cookie is evaluated on each HTTP request. F5 article [K7964](https://support.f5.com/csp/article/K7964) has details about these intricacies.
1. When removing/decommissioning backend servers fronted by a load balancer configured with cookie persistence, keep in mind that clients will retain the persistence cookie that tells the load balancer to send them to a specific node. If not managed correctly, this will tell the load balancer to send clients to one of the nodes that was removed from the pool, which will usually return a TCP reset to the client since the node is no longer part of the load balancer's connection table.

## Story time

A coworker of mine recently stumbled over pitfall #3 in the form of user complaints when they decommissioned two web servers, and ran some potential solutions by me:

1. Tell users to clear their cookie for the website. Far from ideal - getting users to do what you want them to is often harder than getting computers to do what you want them to. However, the message would be simple enough for a helpdesk technician to communicate to the average user as a workaround in a pinch.
1. Change the persistence configuration on the virtual server. The load balancer was configured to present HTTP persistence cookies for the duration of the client's session. This meant that the persistence cookie lifespan was at the mercy of each client's browser to determine what an "HTTP session" was. For most modern browsers, this almost certainly meant "until the browser window is closed." While a time-based cookie expiration could be configured, the existing cookies cached by client browsers still had to be addressed.

In light of those challenges, I proposed the following instead:

- Add the nodes back to the pool in a disabled state. This would allow the load balancer to make a more informed load balancing decision based on the state of the pool members. Depending on the load balancing platform and the pool/node configuration options available, the response could vary from ungraceful (TCP/HTTP error, followed by new load balancing decision and subsequent persistence cookie on page refresh) to seamless (detect that the target node is down, send the client to a healthy node, and return a new persistence cookie).
- Add logging to the virtual server to inspect the target specified in each persistence cookie, and filter by the addresses of the servers to be decommissioned. Then, remove the servers from the pool for good once a week had passed or the observed logs reported zero hits within a 48-hour window, whichever of those two events came first.

The issue ended up being moot because a short conversation with the developers of the web application revealed that the application was stateless, obviating the need for persistence in the first place. :sweat_smile:
