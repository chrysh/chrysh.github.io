---
title: Getting involved
date: '2024-02-17'
categories:
  - Rust
  - Linux Kernel
  - Kernel Driver
  - Community
tags:
  - Rust
  - Linux Kernel
  - Kernel Driver
  - Community
  - Open Source
  - Zulip
---

[<img class="penguin" src="/static/img/rusty_penguin_4.jpeg" alt="Rusty penguin. Created by DALL·E 3." />](https://github.com/Rust-for-Linux/)

# Rust for Linux
The best starting place is probably the web page of the project,
[rust-for-linux.com](https://rust-for-linux.com). This page is meant as a hub of
links, documentations and resources to this topic.  One source that does not
seem to work well is the github [Good first
issues](https://github.com/Rust-for-Linux/linux/contribute) page, which
currently is not up to date, and all good first issues are already done or in
progress by somebody.  On the other hand, the Rust-for-Linux [github
repo](https://github.com/Rust-for-Linux/) is still in use and commits are pushed
to it on a regular basis.


The different branches of the project are explained
[here](https://rust-for-linux.com/branches).  The site on
[Contributions](https://rust-for-linux.com/contributing) gives additional ideas
how to advance the integration of Rust in the Linux Kernel.

The mainline kernel does not merge code unless there is a use case in-tree, and
Rust is no exception there. Therefore, only things that are actually used can be
upstreamed.  When new Rust kernel modules arrive, the features that they depend
on are also added. As an example, workqueues were merged because Rust Binder
will need them eventually.

# Mailing list
You can subscribe to the mailing list
[rust-for-linux@vger.kernel.org](mailto:rust-for-linux@vger.kernel.org) and
receive all the discussions and patches related to Rust-for-Linux, and therefore
stay up to date on everything that happens in this project. The development of
Rust for Linux is happening on this mailinglist, and this is also where you
would send your patches to.

# Zulip
[Zulip](https://rust-for-linux.zulipchat.com/) is a mix between a chat and a
forum, where all kinds of questions about this project can be asked, and will be
answered by other people in the project.
The questions are sorted by topic (e.g.
Networking, Library, Filesystems, Meetings, Modules, etc.). If you ask a
question, you can be sure to have an answer the latest in a few days time.

Zulip is the place which is most active
and a good starting point for beginners to get in contact and maybe find out what
a good first project could be. Many people involved in writing
abstractions/drivers are there, and may have some suggestions for new drivers in
their area of expertise.


# Rust reference drivers
Another good starting point is also having a look at [Rust reference
drivers](https://rust-for-linux.com/rust-reference-drivers), which are drivers
that subsystem maintainers are allowed to introduce in their subsystem in order
to provide example implementations of existing drivers, therefore in this way
bootstrap abstractions. This "live documentation" is also used to prepare the
subsystem for Rust over time, and to see for the maintainers whether the effort
to advantage is worth it for their subsystem.

The first reference driver joining mainline is the one for the [Asix
PHYs](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/net/phy/ax88796b_rust.rs?h=v6.8-rc1).

If you have never seen a Linux
Kernel driver written in Rust, look at this [LWN
article](https://lwn.net/Articles/863459/) comparing a GPIO driver between the C
and the Rust implementation. Also from personal experience, I can say that using
Rust instead of C reduces the lines of code quite a bit.

# Open meetings
If you want to get in touch, also feel free to join one of the [Open
Meetings](https://rust-for-linux.com/contact#open-meeting).
