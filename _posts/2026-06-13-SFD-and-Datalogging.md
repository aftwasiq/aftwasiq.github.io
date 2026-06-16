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

## Datalogging

After getting the SFD running, I decided to go ahead and add datalogging to my current test file in order to create an embedded graph.

We'll start by intializing two different loop handlers (using the C wrapper function equivalents), but make sure we have buffers for them. Since the C Wrapper needs memory sizes and addresses.

```
uint8_t ff_buffer[CPP_CONST_SCRUTINY_C_LOOP_HANDLER_FF_SIZE];
uint8_t vf_buffer[CPP_CONST_SCRUTINY_C_LOOP_HANDLER_VF_SIZE];
```

Then I construct the loop handlers: 

```
scrutiny_c_loop_handler_ff_t *task_100hz_lh = scrutiny_c_loop_handler_fixed_freq_construct(ff_buffer, sizeof(ff_buffer), 100000, task_100hz_lh);
scrutiny_c_loop_handler_vf_t *task_idle_lh = scrutiny_c_loop_handler_variable_freq_construct(vf_buffer, sizeof(vf_buffer), task_idle_lh);
```

*LATER EDIT: So because the loop tasks are seperate tasks that I declared before init, I decided to initialize the tasks as NULL at the start of the program and then assign them the constructs later on during the Init() component*

Oh, and we also need to make sure to allocate a large datalogging buffer. It should be as big as possible. 

`uint8_t scrutiny_datalogging_buffer[4096];`

unfinished todo
