---
layout: post
title: "Mutual TLS"
date: 2022-06-07
---
## :lock: What is mutual TLS?

Mutual TLS (commonly abbreviated "mTLS", and also referred to as "client certificate authentication") is an implementation of the TLS protocol for mutual authentication in a client-server model. Essentially, both the client *and* the server must trust each other to establish a mutual TLS session, as opposed to the default server-only authentication model that TLS is most commonly known for. The primary use cases I've observed for mTLS have been securing business-to-business HTTP traffic in highly regulated industries (such as healthcare and financial information exchange) and enforcement of zero-trust corporate networks.

[IETF RFC 5246 (TLSv1.2), section 7.4.6](https://datatracker.ietf.org/doc/html/rfc5246#section-7.4.6) briefly describes the method by which a client can present its proof of identity (an X.509 certificate) to a server for mutual TLS, if the server prompts for one. The TL;DR is that you configure a server to ask for each client's public key, and the server verifies the client's public key against its PKI during the TLS handshake.

## Configuring mutual TLS

Regardless of platform, there is a common set of requirements to enforce mutual TLS on a server:
- This should go without saying, but the server must be configured to terminate client TLS connections. In other words, server TLS is a prerequisite for mutual TLS, so the server must be configured with its own public/private key pair.
- The server must be configured to issue a client authentication challenge.
- One or more certificate authorities must be specified against which the server should verify the identities of each client. This is usually in the form of a single file containing the concatenated X.509 certificates for the full trust chain of each certificate authority.
  - Some platforms also allow you to specify a CRL file to perform certificate revocation status checking.

## Troubleshooting

The most common mutual TLS issues that I've observed typically arise from misconfigurations. Some of them are honest oversights; others originate from fundamental misunderstandings about how mutual TLS works. 

- The full CA chain is not specified on the server. Most modern browsers are intelligent enough to automatically complete any missing intermediate CA certs in a CA chain, but this isn't a guarantee for mobile browsers and IoT devices.
- Outbound internet access over TCP port 80 is denied, preventing the server from performing CRL checks. Depending on the strictness of the server's TLS implementation, this can either manifest as a long delay while the CRL check fails open, or the TLS handshake simply fails closed and returns a TCP reset.
- Possibly the most bizarre issue I've seen was a customer that had configured their BIG-IP to perform CRL verification against the CRL distribution point of the intermediate CA instead of the client certificate itself. These manifested as generic TLS errors in the BIG-IP local traffic logs, and required an `ssldump` capture to determine that the BIG-IP was requesting the wrong CRL distribution point during the TLS handshake.