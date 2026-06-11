---
layout: post
title: "Getting started with Scrutiny + RTEMS"
date: 2026-05-26 12:00:00 +0000
---
## Preliminary Information
### What is RTEMS?

RTEMS is an open source real-time operating system for embedded systems used in a multitude of applications ranging from spacecraft to experimental physics. 

### What is Scrutiny?

Scrutiny is a C++ instrumentation library which provides the user with a seamless real-time debugging experience, it works via a client GUI and a compiled firmware library which communicate with eachother over any protocol.

### Why bring Scrutiny to RTEMS?

RTEMS currently relies primarily on stop-mode debugging via JTAG tools. Unfortunately, stop-mode debugging actively halts your processor during hardware analysis. This means that it cannot be used to debug a live or real time system that is already running, hence why JTAG based tools are more suitable for startup. Another issue is that JTAG based debugging tools tend to be proprietary or expensive, and ones that are not are often difficult and complex to use. With efforts to bring RTEMS forward as more usable for a greater degree of engineers beyond specific scientific niches, easy to use debugging tools will prove invaluable to the projects future. With Scrutiny, this will be possible, allowing users to debug systems that are already live, aswell as providing a beginner & amateur friendly framework for debugging in the first place.

## Getting Started

### Cross-Compiling RTEMS with Scrutiny

To use Scrutiny with RTEMS for initial development, you'll need to build RTEMS with Scrutiny. This is a relatively straightforward process. 

*todo: add details*

### Building Scrutiny with RTEMS

For this GSoC '26, I've completed the preliminary work of creating a build recipe so that Scrutiny can build with the RTEMS Source Builder as a third party package. That being said, it is currently posted as a draft merge request will be assessed after the completion of the deliverables for this GSoC project. You can view the draft merge [here](https://gitlab.rtems.org/rtems/tools/rtems-source-builder/-/merge_requests/229) for more details.

### Next post: Running Scrutiny with RTEMS (successfully launching the GUI via UART)
