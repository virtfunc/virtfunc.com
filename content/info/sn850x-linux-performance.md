---
title: "WD SN850X Linux performance"
date: 2025-04-18
categories: 
  - "info"
tags: 
  - "linux"
  - "nvme-speed"
  - "sn850x"
---

Here is my benchmark data using KDiskMark, using the standard preset on v3.1.4 with linux kernel 6.14.2.

| Test | Read \[MB/s\] | Write \[MB/s\] |
| --- | --- | --- |
| SEQ1M Q8T1 | 7,169.86 | 6,365.18 |
| SEQ1M Q1T1 | 4,678.79 | 3,891.49 |
| RND4K Q32T1 | 1,367.65 | 1,165.72 |
| RND4K Q1T1 | 75.43 | 269.45 |

![proof](/images/sn850x-linux-performance/perf.png)
