---
layout: post
title: ":clock3: Add-ADGroupMember and Time-Based AD Group Membership"
date: 2022-08-05
---
Did you know that time-based AD group membership has to last for at least one millisecond? Here's how I found out.

## Active Directory Privileged Access Management

The Windows Server 2016 AD forest functional level introduced [Privileged Access Management](https://docs.microsoft.com/en-us/microsoft-identity-manager/pam/privileged-identity-management-for-active-directory-domain-services) (PAM), a set of tools and services to help AD admins improve their security posture via tighter access control. Among these was the ability to grant temporary access to AD groups, with the intent of allowing an admin to reduce the need for long-lived privileged user accounts. To that end, the [`Add-ADGroupMember`](https://docs.microsoft.com/en-us/powershell/module/activedirectory/add-adgroupmember?view=windowsserver2016-ps) PowerShell cmdlet was given a new parameter: `MemberTimeToLive`.

## MemberTimeToLive

If there is one thing that you take away from this article, remember this:
1. **The `MemberTimeToLive` parameter requires a [`TimeSpan`](https://docs.microsoft.com/en-us/dotnet/api/system.timespan?view=net-6.0) object as input.**

This seemingly innocuous statement may come across as patronizingly obvious, especially if you have already read `Get-Help Add-ADGroupMember`, but there are several interesting implications that led to me being hoisted by my own petard.

## How I was hoisted by my own petard

### Lighting the fuse

The team I currently work with maintains a self-service web portal based on Mark Domansky's [WebJEA](https://github.com/markdomansky/WebJEA) project for a wide variety of common requests. One of the neat things about WebJEA is the ability to supply a properly-written PowerShell script to the web application, which is then turned into a web form based on the script's help text and parameters. Given that we had already written a few self-service PAM scripts for select privileged groups, and our forest functional level had recently been raised to Server 2016, I took it upon myself to add time-based group membership to these scripts. The relevant bits of code looked something like this:

```powershell
# Parameters from user input
$startDate = Get-Date '2022/08/01 13:00' 
$endDate   = Get-Date '2022/08/01 13:30'

# Calculate access duration
$timeSpan = $endDate - $startDate

# Add user to group
Add-ADGroupMember -Identity 'GroupName' -Members 'Username' -MemberTimeToLive $timeSpan
```

A few hours of testing and change control later, I merged my changes to our PAM scripts, and moseyed on home.

### Smelling smoke

The first issue manifested about a week later. A number of my teammates had successfully granted their user accounts temporary elevated access via the self-service portal for change tasks, and I had confirmed their membership expiry in our SIEM system. As I sipped my afternoon tea, supremely satisfied with my minor improvement, a clever thought crossed my mind - I had a 30-minute change window coming up the next day at 10 AM that required similar elevated access. *What if I scheduled elevated access for my user account in advance for the exact duration of time that I would need it for?*

```powershell
# PowerShell representation of the values I filled out in the self-service web form
$today = Get-Date
$tomorrow = $today.AddDays(1)
$startDate = Get-Date -Date $tomorrow.Date -Hour 10 -Minute 0
$endDate = $startDate.AddMinutes(30)
```

A few clicks in the web portal later and I was once again headed home for the day.

The next day, attempting to execute my change resulted in a cascade of `Access Denied`-flavored errors in my terminal. A SIEM query revealed that my group membership had expired exactly 30 minutes after I had submitted my self-service request the previous day.

What I failed to realize was that **time-based group membership begins as soon as `Add-ADGroupMember` is executed**. You may recognize that what happened here was **just-in-time (JIT) administration**, meaning that elevated access is granted only as soon as you need it and not a moment before. Adherence to the JIT principle means that `TimeSpan` is the ideal data type for the `MemberTimeToLive` parameter, since a `TimeSpan` contains only a time *interval*, with no identifying dates to anchor the interval between two fixed points in time.

```powershell
PS> $endDate - $startDate | Get-Member -MemberType Property

   TypeName: System.TimeSpan

Name              MemberType Definition
----              ---------- ----------
Days              Property   int Days {get;}
Hours             Property   int Hours {get;}
Milliseconds      Property   int Milliseconds {get;}
Minutes           Property   int Minutes {get;}
Seconds           Property   int Seconds {get;}
Ticks             Property   long Ticks {get;}
TotalDays         Property   double TotalDays {get;}
TotalHours        Property   double TotalHours {get;}
TotalMilliseconds Property   double TotalMilliseconds {get;}
TotalMinutes      Property   double TotalMinutes {get;}
TotalSeconds      Property   double TotalSeconds {get;}
```

After realizing my rookie mistake, one swift commit later and I had moved `startDate` from a parameter to an internal variable, with an additional bit of help text stating that the scripts would grant just-in-time access.[](https://multitwitch.tv/evo/evo2/evo3/evo4/evo5/evo6/evo7/teamsp00ky/playstation) The code now looked like this:

```powershell
# Parameters from user input
$endDate = Get-Date '2022/08/02 13:30'

# startDate is now calculated at script runtime
$startDate = (Get-Date).Date

# Calculate access duration
$timeSpan = $endDate - $startDate

# Add user to group
Add-ADGroupMember -Identity 'GroupName' -Members 'Username' -MemberTimeToLive $timeSpan
```

This time, I went home rather sheepishly at the end of the day, having been gently chided for such a simple oversight by the senior engineer who reviewed my merge request.

### Detonation

A month later, PAM portal usage had spread to other teams, and had become an integral part of our change control process. While refactoring the script to fit other teams' needs, I decided to condense the date calculation:

```powershell
$timeSpan = $endDate - (Get-Date)
```

Considering this a minor change, I was allowed to merge the change with minimal pomp and circumstance.

Then everything broke.

Here's what happened:

```
Add-ADGroupMember: The parameter is incorrect.
```