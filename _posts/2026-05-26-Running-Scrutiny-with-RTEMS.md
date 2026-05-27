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

Taking a basic test file (init.c) from the RTEMS testsuites, we can modify it to build Scrutiny:

```C
/* SPDX-License-Identifier: BSD-2-Clause */

/*
 *  COPYRIGHT (c) 1989-2012.
 *  On-Line Applications Research Corporation (OAR).
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 * 1. Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
 * AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
 * LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
 * CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
 * SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
 * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
 * CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
 * ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
 * POSSIBILITY OF SUCH DAMAGE.
 */

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include <rtems.h>
#include <tmacros.h>
#include <scrutiny.h>

const char rtems_test_name[] = "SCRUTINY DEMO";

static scrutiny::MainHandler scrutiny_handler;
static scrutiny::Config scrutiny_config;

uint8_t rx_buffer[32];
uint8_t tx_buffer[48];

extern "C" {
    rtems_task Init( rtems_task_argument ignored )
    {
        (void) ignored;

        rtems_print_printer_fprintf_putc( &rtems_test_printer );
        scrutiny_handler.init(&scrutiny_config);
        TEST_BEGIN(8_
        printf( "Scrutiny handler initialized & working on sparc/erc31 TEST\n" );
        TEST_END();
        rtems_test_exit( 0 );
    }
}

/* NOTICE: the clock driver is explicitly disabled */
#define CONFIGURE_APPLICATION_DOES_NOT_NEED_CLOCK_DRIVER
#define CONFIGURE_APPLICATION_NEEDS_SIMPLE_CONSOLE_DRIVER

#define CONFIGURE_MAXIMUM_TASKS 1

#define CONFIGURE_RTEMS_INIT_TASKS_TABLE

#define CONFIGURE_INIT_TASK_ATTRIBUTES RTEMS_FLOATING_POINT

#define CONFIGURE_INITIAL_EXTENSIONS RTEMS_TEST_INITIAL_EXTENSION

#define CONFIGURE_INIT
#include <rtems/confdefs.h>

```
### Next post: Running Scrutiny with RTEMS (successfully launching the GUI via UART)
