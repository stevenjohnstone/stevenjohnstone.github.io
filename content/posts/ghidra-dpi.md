---
title: "Ghidra High-DPI on Ubuntu"
date: 2021-02-27T23:47:04Z
---

## Problem

![small fonts](/images/highdpi/scale1-small.jpg#center)

The fonts are scaled so as to be unreadable (for me).

## Solution

Find the file support/launch.properties where you installed Ghidra and edit the line

```
VMARGS_LINUX=-Dsun.java2d.uiScale=1
```

to

```
VMARGS_LINUX=-Dsun.java2d.uiScale=2
```

![larger fonts](/images/highdpi/scale2-small.jpg#center)



