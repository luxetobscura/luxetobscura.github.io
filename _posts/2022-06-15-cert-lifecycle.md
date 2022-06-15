---
layout: post
title: ":recycle: Certificate Lifecycle"
date: 2022-06-12
---
Certificate management is hard. Part of my job is to make it less hard.

## Requirements

* Certificates are valuable assets and should be treated as such. They are an approximation of a securable object's identity, whether that object is a person, machine, or application.
* Why will the service require authentication? Is a certificate the ideal authentication method?
* Who needs to trust the certificate? (This answers who the certificate authority should be, and possibly where the cert should be deployed.)
* Which securable object operates the service? (This answers who needs permissions to the certificate's private key.)
* Where should the certificate be deployed?
* What type of certificate should be used? (This is where you decide the bells and whistles like key algorithm and other extensions.)
* Based on the previous queries, codify these requirements into a configuration file that can be referenced in the future.

## Deployment

* Generate a certificate request.
* Sign the certificate request (possibly optional, depending on your certificate authority).
* Submit the certificate request to the certificate authority.
* Complete the request by installing the returned certificate in the original context (user or machine, mostly a Windows concern).
* Create certificate object in secret management system (KeePass, Bitwarden, Thycotic Secret Server, etc.).
* Deploy certificate to target machines.
* Set private key permissions on target machines.
* Clean up certificate objects from intermediate staging locations (including machine memory), if any.
* Work with the original requestor(s) to validate service functionality.

## Renewal

* Certificate expiration alert triggered by monitoring system.
* Based on thumbprint or serial number, identify the cert and retrieve its configuration file.
* Create a new deployment based on the original config file. A few differences between fresh deploy and renewal:
  * Either increment upon or rotate the original certificate object in your secret management system.
  * Rotate the certificate on targets.

## Retirement

* Mutual agreement between original requestors and system administrators that the service has reached end of life.
* Remove certificates from targets.
* Mark certificate objects as retired in your secret management system.
* If the sole purpose of the targets was to present this single service, decommission the targets.
