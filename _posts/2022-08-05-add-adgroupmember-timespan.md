---
layout: post
title: ":clock: Add-ADGroupMember and Time-Based AD Group Membership"
date: 2022-08-05
---
I learned the hard way that you can't assign temporary membership to an AD principal for a duration of less than a millisecond. Here's how I found out.

## Active Directory Privileged Access Management

The Windows Server 2016 AD forest functional level introduced [Privileged Access Management](https://docs.microsoft.com/en-us/microsoft-identity-manager/pam/privileged-identity-management-for-active-directory-domain-services) (PAM), a set of tools and services to help AD admins improve their security posture via tighter access control. Among these was the ability to grant temporary access to AD groups, with the intent of allowing an admin to reduce the need for long-lived privileged user accounts. To that end, the [`Add-ADGroupMember`](https://docs.microsoft.com/en-us/powershell/module/activedirectory/add-adgroupmember?view=windowsserver2016-ps) PowerShell cmdlet was given a new parameter: `MemberTimeToLive`.

## MemberTimeToLive

There is one thing you should know about the `MemberTimeToLive` parameter:
1. The `MemberTimeToLive` parameter requires a [`TimeSpan`](https://docs.microsoft.com/en-us/dotnet/api/system.timespan?view=net-6.0) object as input.

This seemingly innocuous statement may come across as patronizingly obvious, especially if you have read `Get-Help -Name Add-ADGroupMember`, but there are several interesting implications that led to me being hoisted by my own petard.

## How I was hoisted by my own petard, twice

The team I currently work with maintains a self-service web portal based on Mark Domansky's [WebJEA](https://github.com/markdomansky/WebJEA) project for a wide variety of common requests. One of the neat things about WebJEA is the ability to supply a properly-written PowerShell script to the web application, which is then turned into a web form based on the script's help text and parameters. Given that we had already written a few self-service PAM scripts for select privileged groups, I took it upon myself to further explore time-based group membership. A few hours of scripting, testing, and change control later, I merged my changes to our PAM scripts, and I moseyed on home.

The first issue manifested about a week later. A number of my teammates had successfully granted their user accounts temporary elevated access via the self-service portal for change tasks, and I had confirmed their membership expiry in our SIEM system. As I sipped my afternoon tea, supremely satisfied with my minor improvement, a clever thought crossed my mind - I had a 30-minute change window coming up the next day that required similar elevated access. *What if I scheduled elevated access for my user account in advance for the exact duration of time that I would need it for?* A few clicks later and I was once again headed home for the day.

What I failed to realize was that **time-based group membership begins as soon as `Add-ADGroupMember` is executed**. The next day, attempting to execute my change resulted in a cascade of `Access denied`-flavored errors in my terminal. A SIEM query revealed that my group membership had expired exactly 30 minutes after I had submitted my self-service request the previous day. You may recognize that what happened here was **just-in-time (JIT) administration**. A swift merge request later and I had rewritten the PAM scripts to be explicitly clear that they would grant just-in-time access.

A month later, the second pitfall befell me. At this point, the PAM portal had been adopted into use by a number of other teams, including application operators. 

<WIP>
