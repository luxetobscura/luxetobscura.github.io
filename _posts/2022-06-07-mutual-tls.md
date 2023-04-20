---
layout: post
title: ":lock: Mutual TLS, aka Client Certificate Authentication"
date: 2022-06-07
updated: 2023-04-19
---
I had to set up mutual TLS once. Learning how to do it from scratch sucked, so here's what I wish I knew when I started.

## What is "mutual" TLS, anyways?

If you're familiar with one-way TLS already, mutual TLS basically means that both the client and server have to trust each other as part of the TLS session negotiation process. The main difference is that in addition to the server-side setup required for one-way TLS, the server must also be configured to challenge the client for its public key, and the client must be prepared to provide it.

If you're like me from 4 years ago and you *aren't* familiar with TLS already, mutual TLS (commonly abbreviated "mTLS") is an implementation of the TLS protocol for mutual authentication in a client-server model. Essentially, both the client *and* the server must trust each other to establish a mutual TLS session, as opposed to the default server-only authentication model that TLS is most commonly known for.

The primary use case that I've observed for mTLS has been securing business-to-business (B2B) HTTP traffic in highly regulated industries, such as healthcare and financial information exchange. It's a crucial part of enforcing the zero trust security model required by the regulating bodies governing those industries. More recently, it's also playing an increasing role as a phishing-resistant authentication factor in interactive user authentication workflows.

On occasion, you may also hear the terms "client certificate authentication" or "certificate-based authentication." These refer to the general concept of a signed certificate required for a server to validate the identity of a client. Other authentication methods, such as smart cards, implement certificate-based authentication as well. However, these terms most often refer specifically to mutual TLS authentication.

[IETF RFC 5246 (TLSv1.2), section 7.4.6](https://datatracker.ietf.org/doc/html/rfc5246#section-7.4.6) briefly describes the method by which a client can present its proof of identity (an X.509 certificate) to a server for mutual TLS, if the server prompts for one. The TL;DR is that you configure a server to ask for each client's public key, and the server verifies the client's public key against its PKI during the TLS handshake.

## Configuring mutual TLS

Regardless of platform, there is a common set of requirements to enforce mutual TLS on a server:

- This should go without saying, but the server must be configured to terminate client TLS connections. In other words, server TLS is a prerequisite for mutual TLS, so the server must be configured with its own X.509 public/private key pair.
- The server must be configured to issue a client authentication challenge.
- One or more certificate authorities must be specified against which the server should verify the identities of each client. This is usually in the form of a single file containing the concatenated X.509 certificates for the full trust chain of each certificate authority.
  - Some platforms also allow you to specify CRL distribution points and/or OCSP responders to perform certificate revocation status checking.

## Pitfalls

The most common mutual TLS issues that I've observed typically arise from misconfigurations. Some of them are honest oversights; others originate from fundamental misunderstandings about how mutual TLS works.

- The full CA chain is not specified on the server. Most modern browsers are intelligent enough to automatically complete any missing intermediate CA certs in a CA chain, but this isn't a guarantee for mobile browsers and IoT devices.
- Outbound internet access over TCP port 80 is denied, preventing the server from performing CRL checks. Depending on the strictness of the server's TLS implementation, this can either manifest as a long delay while the CRL check fails open, or the TLS handshake simply fails closed and returns a TCP reset.
- Possibly the most bizarre issue I've seen was an organization that had configured their F5 BIG-IP load balancer to perform CRL verification against *the CRL distribution point (CRLDP) of the intermediate CA*, instead of the CRLDP of the client certificate itself. These manifested as generic TLS errors in the BIG-IP local traffic logs, and required an `ssldump` capture to determine that the BIG-IP was requesting the wrong CRL distribution point during the TLS handshake.
