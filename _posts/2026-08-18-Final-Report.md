---
layout: post
title: "Final Report"
date: 2026-08-18 12:00:00 +0000
---

GSoC now comes to an end, and I learned a lot these past months. I had a great time contributing to RTEMS. That being said, I am not completely done at the time of writing, as my MRs are in review status, and after they will need to be checked/updated as per anything my mentors or other maintainers point out.

## Project Scope

This project encompassed several objectives:

- Prototyping & creation of a two-way UART transport that successfully initializes all Scrutiny functions.
- Identify which metrics to implement and add their implementation.
- Add example initialization code based on earlier prototyping and demo
- Add documentation regarding using Scrutiny with RTEMS
- Build recipe (already written in advance)

However, another deliverable was added on while making instrumentation metrics easier to implement. 

- Add a new CPU usage directive to the Classic RTEMS API
- Add documentation for the new CPU usage directive
- Add testsuite for the new CPU usage directive

## Accomplishments & leftover work

I ended up finishing most of my deliverables. The only thing I could not properly wrap up was sample instrumentation metrics. I did have a basic interrupt count & CPU usage working, however my interrupt count metric was a bit rough, and CPU usage relied on the `rtems_task_get_cpu_usage()` directive which took far longer than expected to work on. However, I'm fine with this tradeoff, because I plan to add instrumentation metrics as incremental additions after GSoC. This API addition was a great learning experience, and I think it was a good use of time, because this directive may benefit RTEMS users, even outside of using Scrutiny.

I'm excited to continue contributing to RTEMS, and Scrutiny aswell. I'll be expanding the Scrutiny integration effort over time, as I said, but also adding support for different transports. 

## Code (currently at time of writing 08/18, it is all either draft or unmerged)

- Adding `rtems_task_get_cpu_usage()` as a new directive to the classic RTEMS API: [MR !1347](https://gitlab.rtems.org/rtems/rtos/rtems/-/merge_requests/1347/diffs)
- Documentation for the `rtems_task_get_cpu_usage()` directive: [MR !248](https://gitlab.rtems.org/rtems/docs/rtems-docs/-/merge_requests/248)
- Scrutiny initialization code under `rtems-examples`: [MR !34](https://gitlab.rtems.org/rtems/rtos/rtems-examples/-/merge_requests/34)
- Scrutiny documentation for installing & running with RTEMS: [MR !252](https://gitlab.rtems.org/rtems/docs/rtems-docs/-/merge_requests/252)
- Scrutiny build recipe *(this was mainly made before GSoC)*: [MR !229](https://gitlab.rtems.org/rtems/docs/rtems-docs/-/merge_requests/252)

## Conclusion 

I'm so glad I've been given the opportunity to work on this project. I feel like I've only scratched the surface to RTEMS and I'm eager to learn more. 

I want to thank my mentor Pier-Yves Lessard, without him, I would be stuck debugging basic stuff for weeks. He has been an incredible help and I had several insightful conversations with him. I learnt a lot.

I also want to thank my other mentors Gedare Bloom, Amar Thakar, and Wayne Thornton for all their help in guiding me throughout the RTEMS-related components of the project. I'd still be lost navigating RTEMS without them. I also really appreciate the input from all other maintainers when I wrote up discussion posts regarding help.
