---
layout: post
title: "Scrutiny Firmware Description (SFD) files & Datalogging"
date: 2026-06-13 12:00:00 +0000
---
## Preliminary Information

This post documents my findings and work during the third week of GSoC '26 with RTEMS. Last post I ending up getting Scrutiny up and running with RTEMS via UART on both QEMU and physical hardware. The QEMU route took some extra effort in networking the Scrutiny server to an emulator. Nonetheless we ended up getting it working. I also discussed a potential issue regarding the Scrutiny server communicating with SIS (Sparc Instruction Simulator) which is a simulator typically used for RTEMS applications.

## Why do we need a "Scrutiny Firmware Description" file?

Without an SFD, scrutiny cannot actually read anything from the firmware that it's instrumenting. The SFD provides Scrutiny with all information regarding the firmware id (a 128 bit hash), a variable map (to store debugging symbols), aliases, and its metadata. You can read more about how it works [here](https://scrutinydebugger.com/guide-instrumentation.html#postbuild-toolchain).

## Generating an SFD 

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

*Add aliases"*
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

## Datalogging

So how do we actually get something now that we have the SFD working?
