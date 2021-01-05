---
title: "Weekend Projects"
date: 2021-01-05T23:27:23Z
---

Brain dump of potential weekend/lockdown projects.

## Threat Modelling Tool

When I make a threat model, I draw ascii art, do doodles, write descriptions but it'd be nicer to have
some tools which allowed me to

1. specify a threat model in a near natural language
2. generate diagrams from the threat models

### Threat Modelling Language

The language would be as close to English as possible. Simple sentences along the lines of

```
The loadbalancer has a certificate
The certificate expires next month
The loadbalancer has 6 backend servers
```

Lexical analysis tools would allow this to be converted into a graph representation of the system. Nodes would be nouns, edges verbs and
there'd be rudimentary understanding of number. Tools which understand the nouns and verbs could be build on top of this to, for example,
suggest OWASP rules to check.

### Graphical Representation

One such tool built on the graph representation could render a nice graphical representation of the system. Output like [this tool](https://github.com/mingrammer/diagrams) creates but with more icons for threats would be just the thing.

Integrations with Hugo (for blogging) and vscode would allow developers to quickly describe a system and threats and generate diagrams.

## IJON for Golang

I've [experimented](https://github.com/stevenjohnstone/afl-lua#51-annotations) a bit with creating a fuzzer for Lua with some advanced features like [IJON](https://github.com/RUB-SysSec/ijon). I'd like to add similar functionality to [go-fuzz](https://github.com/dvyukov/go-fuzz).

## Fuzzers with Novel Guidance Schemes

I'd like to experiment with fuzzing by using signals other than code coverage. It'd be useful when studying black box embedded systems to be able to
detect new coverage by

* power draw
* EM radiation/noise
* temperature changes

In systems running on personal computers, signals like

* cache statistics
* memory usage
* stack size
* heap statistics
* file descriptor usage

may be of interest.


