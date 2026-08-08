---
layout: post
title: "Exposing Metrics from RTEMS to Scrutiny: CPU Usage"
date: 2026-07-30 12:00:00 +0000
---

## Getting Started
Over the past couple weeks I've been working on adding a new addition to the RTEMS classic API in order to make instrumenting CPU usage easier. 

This expands the scope of my GSoC project significantly, and is a very big addition, so I may need to postpone the instrumentation of stack/heap usage until *after* I finish the remaining integration work (example testsuite + documentation) instead of before. 

I'm no particularly worried about instrumenting stack & heap usage as these can be aspects that can be addeed to the integration work as issues even after GSoC. What matters most is that Scrutiny is able to run with RTEMS easily for any user- and clearing up my current CPU Usage API addition (which took longer than I initially anticipated)

## Merge Request

Since most of the work these couple weeks has been far more back and forth I hadn't had too much time to lay out the exact reasoning and development choices of what I'm doing. Hopefully the current open merge request will suffice to explain most of what I've been doing [here](https://gitlab.rtems.org/rtems/rtos/rtems/-/merge_requests/1347)

This MR adds a new directive following a discussion on instrumenting a numeric CPU usage value.

The directive is based off the helper function getrusage_RUSAGE_THREAD from the POSIX function getrusage(). Instead of being hardcoded to the self_id however, it takes a valid rtems_id and fills a timespec struct instead of a rusage struct.

## Details

I will elaborate more on how this will make instrumentation to Scrutiny easier within the coming days. Right now (while the MR is pending approval) I will shift over to adding documentation, writing the example testsuite, finalizing the build recipe, and integrating postbuild activity to waf.
