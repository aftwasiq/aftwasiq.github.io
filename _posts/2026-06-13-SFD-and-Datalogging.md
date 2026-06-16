---
layout: post
title: "Scrutiny Firmware Description (SFD) files & Datalogging"
date: 2026-06-13 12:00:00 +0000
---
## Preliminary Information

This post documents my findings and work during the third week of GSoC '26 with RTEMS. Last post I ending up getting Scrutiny up and running with RTEMS via UART on both QEMU and physical hardware. The QEMU route took some extra effort in networking the Scrutiny server to an emulator. Nonetheless we ended up getting it working. I also discussed a potential issue regarding the Scrutiny server communicating with SIS (Sparc Instruction Simulator) which is a simulator typically used for RTEMS applications.

## Why do we need a "Scrutiny Firmware Description" file?

An SFD provides Scrutiny with all information regarding the firmware id (a 128 bit hash), a variable map (to store debugging symbols), aliases, and its metadata. You can read more about how it works [here](https://scrutinydebugger.com/guide-instrumentation.html#postbuild-toolchain).

## Generating an SFD then using CMake to automate the process

First we need to recompile our test program with the ability to produce an .elf file. This will be crucial as we cannot create an SFD without it.

Then we run the following in order: 

*Create a variable map:*

`scrutiny elf2varmap scrutiny_demo.elf --output .`

*Generate a firmware id:*

`scrutiny get-firmware-id scrutiny_demo.elf --output .`

*Tag a firmware id to an .elf:*

`scrutiny tag-firmware-id scrutiny_demo.elf scrutiny_demo_tagged.elf`

*Write metadata:*

`scrutiny make-metadata --output x --project-name RTEMS_DEMO --version "V1"`

*Add aliases:*

`scrutiny add-alias . --file alias.json`

*Create the SFD:*

`scrutiny make-sfd . scrutiny_demo_v1.sfd`

*Install it (It is crucial that this is installed in the same directory where you run the server)*

`scrutiny install-sfd scrutiny_demo_v1.sfd`

And as we can see, our firmware finally has an SFD:

<img width="1919" height="131" alt="image" src="https://github.com/user-attachments/assets/21c98e51-6e99-42fc-8cee-e8514ba3ce4e" />

And over on the GUI..

<img width="431" height="31" alt="image" src="https://github.com/user-attachments/assets/f91f457e-5c79-44ac-87ae-7306d3504c12" />

(i forgot to add a version to the metadata when taking this screenshot)

HOWEVER, we will need to make this process more efficient via CMakeLists.txt. The Scrutiny documentation specifies the process for this. Originally I had plans to integrate this entire post-build process through the RSB (RTEMS source builder). Since I already have a working Scrutiny package under it in my personal fork. 

Unfortunately I don't think this is possible, because generating the SFD is post-build activity. The package I have and the system (RSB) that builds it is only a one time build. It cross compiles the Scrutiny library and builds it for the BSP specified. Generating the SFD is a seperate process that comes with linking the demo file. This is something I may have to bring up with mentors from the RTEMS side to see if there is a way to integrate this into the RSB.

Regardless, to automate this process I am using the following CMakeLists.txt (I also had to create a toolchain.cmake to store my previous compile statement). Normally I would build through waf, but I also need to talk with my mentors on the RTEMS side to see if I can integrate a post-build process into waf. So for the time being I'll manually compile the demo file and generate its SFD through a simple CMakeLists:

```
project(scrutiny_demo C CXX)
set(RTEMS_BSP_PATH $ENV{HOME}/gsoc/quick-start/rtems/7/sparc-rtems7/leon3)
include_directories(
    ${RTEMS_BSP_PATH}/lib/include
    ${RTEMS_BSP_PATH}/lib/include/cwrapper
    $ENV{HOME}/gsoc/quick-start/src/rtems/testsuites/support/include
)
link_directories(${RTEMS_BSP_PATH}/lib)
add_definitions(-DSCRUTINY_ENABLE_DATALOGGING=1)
add_executable(scrutiny_demo scrutiny_demo.c)

target_link_libraries(scrutiny_demo
    scrutiny-cwrapper
    scrutiny-embedded
    rtemstest
    stdc++
)

include(${RTEMS_BSP_PATH}/lib/cmake/scrutiny/scrutiny-config.cmake)

scrutiny_postbuild(scrutiny_demo
    INSTALL_SFD
    METADATA_PROJECT_NAME "RTEMS_DEMO"
    METADATA_VERSION "1"
    CPPFILT sparc-rtems7-c++filt
)
```

And it works.

## Datalogging

After getting the SFD running, I decided to go ahead and add datalogging to my current test file in order to create an embedded graph.

We'll start by intializing two different loop handlers (using the C wrapper function equivalents), but make sure we have buffers for them. Since the C Wrapper needs memory sizes and addresses.

```
uint8_t ff_buffer[CPP_CONST_SCRUTINY_C_LOOP_HANDLER_FF_SIZE];
uint8_t vf_buffer[CPP_CONST_SCRUTINY_C_LOOP_HANDLER_VF_SIZE];
```

Then I construct the loop handlers: 

```
static scrutiny_c_loop_handler_ff_t *task_100hz_lh = NULL;
static scrutiny_c_loop_handler_vf_t *task_idle_lh = NULL;
```

*Why did I split them up instead of declaring them in one line like I did last time with the main handler? Now we're using lot's of tasks for datalogging, and so I can't declare everything right in the Init(). At least for now, since this demo code is currently experimental.*

```
task_100hz_lh = scrutiny_c_loop_handler_fixed_freq_construct(ff_buffer, sizeof(ff_buffer), 100000, "task_100hz_lh");
task_idle_lh = scrutiny_c_loop_handler_variable_freq_construct(vf_buffer, sizeof(vf_buffer), "task_idle_lh");
```

Oh, and we also need to make sure to allocate a large datalogging buffer. It should be as big as possible. 

`uint8_t scrutiny_datalogging_buffer[4096];`

We need to create a `task_idle()`. We then move the contents of our while loop (which is elaborated on in the last post) into this task. Make sure to remove the while loop. There are some slight changes, before we had a decleration of clock_gettime() with CLOCK_MONOTONIC to get the timestamp before the while loop, and another one in it. This time we only need to declare it once and set `    static uint32_t last_timestamp = 0U;` at the start of the function

We then need to create a `task_100hz()` that simply processes the task without a timestamp.

Here, now we need to change it up by hooking those tasks up to RTEMS tasks entrypoints. We make two rtems_tasks (`task_idle_rtems` and `task_100hz_rtems`, although the names should be _entrypoint instead of _rtems now that I think about it at the time of writing). Both these tasks run a while loop which calls the task and then runs `rtems_task_wake_after()`. The idle tasks uses RTEMS_YIELD_PROCESSOR as a parameter and the 100hz one uses 100ms.

*later edit: We have several functions running for this, I am considering ways to integrate them all into a minimal amount. Expect future changes to the structure of how I layed things out here. Please remember this code is like a rough draft at the moment.*

Okay. Now we have the tasks ready, now to start them. I took a look at a testsuite called ticker.c and basically used the exact same implementation and it worked. So I won't change it for now, since it's working well. In short, I simple create arrays for the tasks ids and names, initialize the clock with the time of day, build the task names, create the tasks, and then start them. This is all in the Init() function.

And... it works!

<img width="488" height="30" alt="image" src="https://github.com/user-attachments/assets/b719d5bd-9c47-4df5-8b11-896343723e11" />

As you can see here, it says "Debugger: Standby". But we want more than that, let's get to generating an embedded graph.

