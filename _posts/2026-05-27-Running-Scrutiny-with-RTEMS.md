---
layout: post
title: "Running Scrutiny with RTEMS via UART"
date: 2026-06-10 12:00:00 +0000
---

## Preliminary Information
This post documents my findings and the work I've done through the first two weeks of GSoC '26. I've taken the first big step towards the completion of my first deliverable (out of four), which is a working UART transport. 

The first order of business is to get the Scrutiny server handshaking with the firmware. For the sake of this blog I'll be going along the QEMU route, so that anybody following along or observing can follow along more easily. 

## Getting started 
We will setup UART through the termios layer that RTEMS provides (since it is a POSIX-compliant OS).

For my current demo, I initalized buffers with the size of 248 bytes, for flexible testing. I am using the C wrapper provided by Scrutiny.

We'll need to open UART2 via the open() system call (not UART1, the reason is because UART1 is used by the operating system, and ideally we do not want to supress the console):

```
int32_t UART2 = open( "/dev/console_b", O_RDWR | O_NONBLOCK | O_NOCTTY );
```
*This example assumes the leon3 BSP. Open "/dev/console_b" with the name of the UART stream your BSP provides if it is different. On real hardware (STM32F4 example), you'd use `dev/ttyS1`.*

Disclaimer: As I said, if you are using QEMU which this post entails, you may not be able to open a second UART. Some chips that are emulated by QEMU only provide one UART. In this case, purely for testing reasons, you will need to supress the console in order to let Scrutiny handshake with the firmware. Using this approach on real hardware is dangerous, it should only be used for testing and integration efforts, which I am pursuing right now. In order to supress the console, you'll need to enter raw mode. You will also need to supress the tmacros defines and functions if this is a test file

Back on track with the UART transport. Now that we've opened UART, we'll need to hook it up to Scrutiny and initalize the MainHandler & ConfigHandler.

Two buffers need to be initialized for the main handler and the config buffer. They use specific definitions written here:

```
uint8_t main_handler_buffer[CPP_CONST_SCRUTINY_C_MAIN_HANDLER_SIZE];
uint8_t config_buffer[CPP_CONST_SCRUTINY_C_CONFIG_SIZE];
```

Then these structs will be initialized in order, the scrutiny config struct (`scrutiny_c_config_t *config`), which takes a construct function as a parameter (`scrutiny_c_config_construct()`). 

Then we set its buffers, adding the scrutiny_rx & scrutiny_tx buffers we initalized earlier through `scrutiny_c_config_set_buffers`. 

We then set up the main handler (`scrutiny_c_main_handler_t *scrutiny_handler`) struct and assign it its specific construct parameter (`scrutiny_c_main_handler_construct()`). We finish off with `scrutiny_c_main_handler_init(scrutiny_handler, config);`.

## Using CLOCK_MONOTONIC and clock_gettime() for timestamps

Then we'll need to hook up a clock to Scrutiny. Scrutiny typically uses microseconds as a timestep. We will be using clock_gettime() with CLOCK_MONOTONIC. More on that choice shortly.

```
clock_gettime(CLOCK_MONOTONIC, &ts);
uint32_t last_timestamp = (ts.tv_sec * 1000000) + (ts.tv_nsec / 1000);
```

## Sending data over UART

We'll need to read from the actual UART stream from the firmware. The earlier rx & tx buffers were initialized for Scrutiny internal functionality. We will need a seperate to call read() with, so another rx & tx buffer for reading the UART stream. 

Then we establish a primary while() loop which will run indefinitely till the program ends. It will call clock_gettime() again with CLOCK_MONOTONIC. The first clock_gettime() gets the current time, and this one gets the next one, so we can have an accurate timestamp. After calculating the time_delta (the current timestamp minus the last_timestamp) we can hook it up to             `scrutiny_c_main_handler_process(scrutiny_handler, time_delta * 10U);`. 

Before that however, we need to actually read the data from the UART stream. Using read():

`ssize_t bytes = read(UART2, in_buffer, sizeof(in_buffer));`

*Why use ssize_t? Because read() can return -1 in the case of error. size_t only takes unsigned integer values*

Then we will recieve the data with `scrutiny_c_main_handler_receive_data();`

Then we will send the data to the scrutiny server. We first check if there are bytes to send with: `scrutiny_c_main_handler_data_to_send();`, If there are, we use `scrutiny_c_main_handler_pop_data()` to pop data from the scrutiny_handler and send over the write() syscall with:

`write(UART2, out_buffer, sent)`

At the end of the while make sure to set the last_timestamp to the timestamp otherwise the loop will never increase the timer.

## Results?

I've saved my file as `scrutiny_demo.c`. For this example using QEMU, I compile against the leon3 BSP with my rtems-scrutiny package, on real hardware I compiled against the stm32f4 BSP. Then we use the following QEMU command:

`qemu-system-sparc -M leon3_generic -m 64M -kernel scrutiny_demo.exe -serial tcp:localhost:5678,server=on,wait=on`

Then you must run socat (socket cat) which is a tool for connecting network streams. Although we wrote a UART transport, since we're emulating this process, we need to do this to connect QEMU and the scrutiny server.

`socat UDP4-LISTEN:1234,fork TCP:localhost:5678`

Then run the scrutiny server:

`scrutiny server --config config.json`

And... it works!

<img width="1919" height="335" alt="image" src="https://github.com/user-attachments/assets/0cb00514-91e1-4251-b3c6-4baed1e1f688" />

On the Scrutiny GUI, we can see we've succesfully connected:

<img width="510" height="32" alt="image" src="https://github.com/user-attachments/assets/29bbc572-7810-4302-8529-68e4f95a83ca" />

## Why QEMU? Why not SIS?

SIS (Sparc Instruction Simulator) is a commonly used simulator in RTEMS for the sparc family BSPs. I originally planned to run Scrutiny for emulation via this simulator. However, the (simulated) firmware was not able to handshake with SIS. Using socat I exposed a PTY (psuedo terminal) to SIS, though despite my best efforts I could not get SIS to handshake with the Scrutiny server.

The bytes the Scrutiny server sends as a DISCOVER request do not get recieved by the simulated firmware and the firmware responds back with garbage (which we assume, but on further research I found that it sends back a frame delimiter).

This may warrant an issue afer or during GSoC in order to fix. During the 06/10 meeting, one of my mentors, Gedare, mentioned that due to Scrutiny being an optional debugger, it is fine that it does not work on every single simulator. For the time being QEMU + physical hardware should suffice, but I would be happy to work on this as a seperate issue after GSoC.

## Next steps:

Next, we will need make changes to the rtems-scrutiny package I currently have made, and add the ability to create an SFD (Scrutiny Firmware Description) file. This file will provide us with the ability to set the debugging symbols in the firmware so we can actually expose instrumentation metrics from RTEMS.

After that, we will need to actually expose instrumentation metrics from RTEMS to Scrutiny. This may involved creatings new functions that will read CPU internals and hardware information from RTEMS that are not currently available.



