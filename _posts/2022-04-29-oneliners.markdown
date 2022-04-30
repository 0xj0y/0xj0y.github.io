---
layout: post
title:  "Important One liners"
date:   2022-04-29 15:30:00 +0530
category: Notes
tags: Notes
---

1. Print all ports comma separated in one line

```bash
cat ports.nmap | grep -i open  | awk -F\/ '{print $1}' | tr '\n' ',' | sed 's/,$//g'
```