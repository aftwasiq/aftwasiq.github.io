---
layout: post
title: "Running Scrutiny with RTEMS via UART"
date: 2026-06-10 12:00:00 +0000
---

## Preliminary Information
This post documents my findings and the work I've done through the first two weeks of GSoC '26. I've taken the first big step towards the completion of my first deliverable (out of four), which is a working UART transport. 

The first order of business is to get the Scrutiny server handshaking with the firmware. For the sake of this blog I'll be going along the QEMU route, so that anybody following along or observing can follow along more easily. 

## Setting up UART 
We will setup UART through the termios layer that RTEMS provides (since it is a POSIX-compliant OS).

For my current demo, I initalized buffers with the size of 248 bytes, for flexible testing. I am using the C wrapper provided by Scrutiny.

We'll need to open UART2 via the open() system call (not UART1, the reason is because UART1 is used by the operating system, and ideally we do not want to supress the console):

```
int32_t UART2 = open( "/dev/console_b", O_RDWR | O_NONBLOCK | O_NOCTTY );
```
*This example assumes the leon3 BSP. Open "/dev/console_b" with the name of the UART stream your BSP provides if it is different. On real hardware (STM32F4 example), you'd use `dev/ttyS1`.*

Disclaimer: As I said, if you are using QEMU which this post entails, you may not be able to open a second UART. Some chips that are emulated by QEMU only provide one UART. In this case, purely for testing reasons, you will need to supress the console in order to let Scrutiny handshake with the firmware. Using this approach on real hardware is dangerous, it should only be used for testing and integration efforts, which I am pursuing right now. In order to supress the console, you'll need to enter raw mode.

Back on track with the UART transport. Now that we've opened UART, we'll need to hook it up to Scrutiny and initalize the MainHandler & ConfigHandler.

Two buffers need to be initialized for the main handler and the config buffer. They use specific definitions written here:

```
uint8_t main_handler_buffer[CPP_CONST_SCRUTINY_C_MAIN_HANDLER_SIZE];
uint8_t config_buffer[CPP_CONST_SCRUTINY_C_CONFIG_SIZE];
```

Then these structs will be initialized in order, the scrutiny config struct (`scrutiny_c_config_t *config`), which takes a construct function as a parameter (`scrutiny_c_config_construct()`). Then we set its buffers, adding the scrutiny_rx & scrutiny_tx buffers we initalized earlier through `scrutiny_c_config_set_buffers`. We then set up the main handler (`scrutiny_c_main_handler_t *scrutiny_handler`) struct and assign it its specific construct parameter (`scrutiny_c_main_handler_construct()`). We finish off with `scrutiny_c_main_handler_init(scrutiny_handler, config);`.

Then we'll need to hook up a clock to Scrutiny. Scrutiny typically uses microseconds as a timestep. We will be using clock_gettime() with CLOCK_MONOTONIC. More on that choice shortly.

```
clock_gettime(CLOCK_MONOTONIC, &ts);
uint32_t last_timestamp = (ts.tv_sec * 1000000) + (ts.tv_nsec / 1000);
```

We'll need to read from 

