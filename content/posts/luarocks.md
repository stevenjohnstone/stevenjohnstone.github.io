---
title: "Luarocks MITM"
date: 2021-02-11T13:32:36Z
draft: false
categories:
    - security
    - lua
    - supply chains
---


## Background

[LuaRocks](https://luarocks.org/) is a package manager for [Lua](https://www.lua.org/). The executable "luarocks" allows users to

* download Lua packages and dependencies from luarocks.org (the default)
* upload packages to luarocks.org

Several months ago [I found a man-in-the-middle vulnerability](https://github.com/luarocks/luarocks/issues/1215) and reported it to the
LuaRocks project via Gitter. I was directed to create a public issue on github. At the time of writing, there's no CVE assigned by the project or github security
notification.

The bug is that while TLS is used to make connections to luarocks.org, then certificate returned by the site is not checked. This step is crucial in establishing the identity of
luarocks.org and without it, an attacker who can intercept and
modify traffic can impersonate the LuaRocks server.

## Severity

To give an indication of the severity of the bug, let's breakdown all the opportunities it presents to an attacker.

### Scenario #1: Victim is a consumer of LuaRocks

Our victim uses LuaRocks to install packages. If an attacker can be a man-in-the-middle anywhere between the victim and luarocks.org, then
requested packages can be replaced with malicious versions which can do some pretty nasty stuff. For example, luarocks will run
arbitrary commands specified in a [rockspec](https://github.com/luarocks/luarocks/wiki/Rockspec-format) file. Here's an example of luarocks deleting a directory:

```shell
➜  foo git:(master) ✗ cat foo-scm-1.rockspec 
package = "foo"
version = "scm-1"
source = {
   url = ""
}
description = {
   license = "*** please specify a license ***"
}
dependencies = {}
build = {
   type = "command",
   modules = {},
   build_command = "rm -rf bar"
}
➜  foo git:(master) ✗ ls
foo-scm-1.rockspec
➜  foo git:(master) ✗ mkdir bar #this will get deleted
➜  foo git:(master) ✗ luarocks make --local        
rm -rf bar
foo scm-1 is now installed in /home/vscode/.luarocks (license: *** please specify a license ***)

➜  foo git:(master) ✗ ls
foo-scm-1.rockspec
```

You're only limited by your imagination and the linux permissions available (which may be root, the "--local" flag is not the default).
My imagination suggests stealing ssh keys, installing backdoors etc.

If the victim is in the business of creating software and uses LuaRocks as part of a software supply chain, then things get really interesting. It's possible to use this vulnerability to inject malicious code into software products turning this vulnerability for our
hapless victim into a security problem for their customers, constituents, patients, students etc (not all software is written for corporations).

### Scenario #2: Victim is a Lua package maintainer

As a man-in-the-middle, the attacker can

* steal the victim's luarocks API key
* modify package uploads in-flight

This would allow an attacker to attack all the consumers of the
victim's packages.

## Can This Really Be Exploited?

TLS and certificate checking exist for a reason. It's not just about encryption for privacy; [it's about establishing the identity of the
remote party](https://docs.google.com/document/pub?id=1roBIeSJsYq3Ntpf6N0PIeeAAvu4ddn7mGo6Qb7aL7ew). There are many points where an attacker could redirect your traffic to a malicious server (be a MITM). Any party on the path
between the victim and the server can act as a man-in-the-middle: I'd suggest thinking twice about using an open wifi network with LuaRocks
before this is fixed. An attacker who can influence DNS responses for luarocks.org can redirect a victim without needing to be a traditional
man-in-the-middle. There are many ways this can happen: compromise of a local DNS resolver, DNS cache poisoning, DNS misconfiguration,
compromise of the DNS registrar account for luarocks.org etc. Attackers can get quite creative about chaining together exploits to make something
greater than the sum of the parts.

## LuaRock's Response

I've been on the receiving end of bug reports when running
a [PSIRT](https://www.first.org/standards/frameworks/psirts/psirt_services_framework_v1.1). I've also reported my fair share
of bugs to vendors so I have a good idea of indicators of a
mature product. Let's start with the basics.

> Who do I contact if I find a security issue?

The project doesn't have a prominent set of instructions for
reporting security bugs. If the bug finder doesn't know who
to report to, then coordinated responses are difficult. In this
case I asked on the project's gitter and was told to create a
public issue on github.

A simple change LuaRocks could make is to create a [security policy](https://github.com/luarocks/luarocks/security/policy) on github. They could also add a [security.txt](https://securitytxt.org/) to luarocks.org to point security researchers to their policy.

> What should the policy be?

I have strong opinions on that but they aren't really relevant.
The project is run by volunteers and they can do whatever they
like: they are free to just shrug off serious vulnerabilities. If
that's the case, then it's probably best to state that clearly so
users can choose their own risk.
