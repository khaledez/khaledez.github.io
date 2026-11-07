---
title: "The Shared Cache"
slug: the-shared-cache
date: 2026-11-07T10:00:00+02:00
kicker: "A bunix novel"
description: "Eight days, one operating system compiled from source, and the slow discovery that the hard part is not the code but the flow of work."
cast:
  - name: Me
    role: building bunix, Buna Linux
  - name: Claude
    role: the colleague in the terminal, who would later forget the first five days
  - name: The Mac
    role: where it started, kept awake all week
  - name: pro13
    role: the build host, which also runs CI machines that must never be touched
  - name: The testbed
    role: a real laptop that dual-boots Windows, and belongs to the family first
  - name: Nawa
    role: my own AI agent, which once let Claude in through the back door
  - name: Fly
    role: a 16-core machine in Frankfurt, rented by the hour for one job
---
{{< disclaimer >}}This post was written by Claude Code based on the experience building bunix. 
<br>
I gave it the following prompt:<br>
<code>
  I would love to have a summary of what we've done since the beginning of this session written in a story similar to
    the Phoenix Project novel by Gene Kim</code>
<br>
Here we go ...
{{< /disclaimer >}}

<nav class="contents" aria-label="Contents">
  <a href="#part-1"><span>Part 1</span>Bootstrap · 31 Oct – 2 Nov</a>
  <a href="#part-2"><span>Part 2</span>Real hardware · 3 – 4 Nov</a>
  <a href="#part-3"><span>Part 3</span>A desktop · 5 – 6 Nov</a>
  <a href="#part-4"><span>Part 4</span>Flow · 6 – 7 Nov</a>
  <a href="#ledger"><span>Ledger</span>The week in numbers</a>
</nav>

{{< part id="part-1" kicker="Part one" title="Bootstrap" >}}Saturday 31 October to Monday 2 November{{< /part >}}

{{< stamp >}}Saturday, 31 October · 14:57 · the Mac{{< /stamp >}}

### Make it boot

It began with something small. I had minisys, a Linux built from source that until then had only lived as a container image on my Mac. I wanted to hold it in my hands.

<q class="user">I want to create a bootable ISO from minisys using limine bootloader, and systemd dns and network stack loaded so I can reach the internet.</q> Bash would be `/bin/sh`, fish would stay my shell, and curl would prove the network worked. An hour later: <q class="user">what about TLS certificates?</q> Then: <q class="user">add wifi support too.</q>

That's how it goes. You ask for a boot disk and by evening you have certificates, iwd and a list of tools. By the next morning the list had ripgrep, neovim, htop, util-linux and BIND on it. The Wi-Fi firmware came off it, because I'd supply firmware per device.

{{< stamp >}}Sunday, 1 November · 04:27 – 08:42{{< /stamp >}}

### Choosing the ground

The early hours of Sunday were all decisions. BoringSSL or OpenSSL? <q class="user">keep openssl.</q> Limine or something signed? <q class="user">replace limine with grub. I want a signed secureboot version so I can boot it without issues.</q>

Then the real question. Everything we'd built by trial and error suggested we needed a better way to describe a system. Nix came up. <q class="user">I'm not talking about structuring the system as NixOS, just the build system. I want to keep the layout and structure compliant with LSB and UNIX.</q> And then I wrote down what I actually wanted, in one line I'd come back to all week: <q class="user">FHS system from source, built reliably and will be continously built as patches and updates to the dependent packages are added.</q> A few minutes later I made it stricter: the UAPI group's filesystem hierarchy.

At 08:39 it got a name. <q class="user">will go with BuildStream. The system name will be bunix (Buna Linux). I prefer to bootstrap our own toolchain.</q> Three minutes later: <q class="user">start the BuildStream pilot with the toolchain bootstrap.</q>

{{< lesson >}}Write the goal in one sentence before the first line of code. Everything after it gets measured against that sentence.{{< /lesson >}}

{{< stamp >}}Sunday · afternoon · Rosetta, then pro13{{< /stamp >}}

### A shell more modern

The bootstrap ran in x86_64 containers on my Apple-silicon Mac, translated by Rosetta. Stage three, the native compiler, stopped with a configure error: it <q>requires a shell more modern</q>. <q class="user">why did we see "no modren shell" error?</q>

Because Rosetta doesn't implement the `faccessat2` system call, and our new glibc had dropped its fallback for kernels that old. So `test -x /` returned false, and configure decided the shell was broken. The workaround meant a full rebuild. Then bubblewrap failed with <q>Function not implemented</q>. Then mounting `/proc` failed.

At 14:10 I stopped fighting it. <q class="user">if the problem is rosetta, ssh into pro13.home.arpa and do the build their.</q> Builds on pro13, a real x86_64 machine, ran about four times faster. It has been the build host ever since.

The day had one more puzzle. GCC failed with a preprocessor error that made no sense. The cause was a bug in gawk 5.4.1 that only appears when gawk is built without the MPFR library. Claude proved it by building gawk both ways, and we pinned gawk 5.3.2 in the bootstrap.

<q class="user">keep this mac up as long as this session is not done.</q> The Mac was kept awake from then on, for the rest of the week.

{{< lesson >}}When the environment keeps fighting you, change the environment. Moving to a native host made every later fix possible.{{< /lesson >}}

{{< stamp >}}Monday, 2 November · pro13 and QEMU{{< /stamp >}}

### An operating system, and how to deliver it

By Monday the toolchain stood up on its own: glibc 2.44, GCC 16.2, and Rust 1.99.1, whose build took just under two hours and had to run twice because of an rpath bug. A small program printed <q>hello from bunix Rust</q>. Then came step four: the kernel built in 16 minutes and the ISO in 24 seconds. In QEMU it booted in under two seconds, using 196 MB of memory.

That evening I asked the question that shaped everything after it. I wanted to download the toolchain onto a running system, securely, in a way that would later become the system's own update mechanism. The answer was systemd-sysupdate: signed images for `/usr`, and system extensions layered on top of it.

Signing came next. <q class="user">I hate GnuPG, what other modern and secure ways to sign packages/files that can be used with systemd-sysupdate.</q> We kept the OpenPGP format, which sysupdate needs, and dropped the GnuPG program. Sequoia's `sqop` signs and `sqv` verifies. A tiny `gpg` wrapper accepts only `--verify` and hands the job to sqv.

Then I asked whether these layers lived in memory or on disk. <q class="user">wait, so the layers are in memory not on disk?</q> They're on disk, overlaid on `/usr`. It's a good question to ask before you put a whole desktop in one.

{{< part id="part-2" kicker="Part two" title="Real hardware" >}}Tuesday 3 November to Wednesday 4 November{{< /part >}}

{{< stamp >}}Tuesday, 3 November · 06:03 · the testbed{{< /stamp >}}

### The photograph

QEMU had proved the updates and the rollback: a deliberately broken 0.1.2 used up its three boot tries, and the machine went back to 0.1.1 by itself. So I put the ISO on a real laptop, one that dual-boots Windows, and installed bunix into an existing partition.

It wrote the filesystem and stopped: <q>od cannot skip past end of combined input</q>. The installer read the Secure Boot state with `od`, and the firmware's variable files report a size of zero. The fix was to read through `head` instead of skipping.

Then the boot failed outright. I took a photo of the screen with my phone. <q class="user">boot failed, check ~/Downloads/screenshot.JPG.</q> macOS wouldn't let Claude read my Downloads folder, so I moved the file myself. The photo said <q>Failed to open \EFI\bunix\grubx64.efi</q>. This firmware doesn't pass the boot loader's name along to shim, so shim falls back to looking for that exact file name.

So systemd-boot now ships under GRUB's file name. <q class="user">why do you call it grubx64.efi if it is systemd-boot?</q> Because that's the name the firmware will load. At 06:30: <q class="user">it booted, systemd-boot menu shows bunix and windows.</q> Eight minutes later: <q class="user">yes, drop grub and BIOS.</q>

{{< lesson >}}The emulator tells you whether the design works. Only real hardware tells you whether it works on the hardware people actually own.{{< /lesson >}}

{{< stamp >}}Tuesday · 07:43 – 11:10 · GitHub{{< /stamp >}}

### Going public

To test updates on a real machine, the updates had to live somewhere. iCloud was on the table. <q class="user">you have my permission create the repo and publish the release.</q> [bunasys/bunix](https://github.com/bunasys/bunix) appeared on GitHub, and 0.1.0 and 0.1.1 went up.

The first update showed nothing. It had run before 0.1.1 finished uploading. Then: <q class="user">now I see the new version!</q> By late morning Rust and Swift had their own extensions, and the verity partitions had shrunk from 64 MiB to 16. <q class="user">updated and rebooted, rustc and swift both work.</q>

{{< stamp >}}Tuesday · 11:23 → night{{< /stamp >}}

### The vision, and the back door

Then came the long message. Graphics. The Buna logo while it boots. Touch screens. Hyprland. A normal user, not just root. A graphical path from boot through install. Arabic, from day one. And: <q class="user">make sure we're using the latest LTS kernel BTW.</q>

The logo appeared, squashed. <q class="user">the logo was not isometric, it looked shrinking vertically.</q> The graphics driver was loading too late; moving it into the initrd fixed the shape.

That night I did something unusual. I ran Nawa, my own AI agent, with a DeepSeek model on the testbed and asked it to install an SSH server, so Claude could get in and check things for real. <q class="user">you can check nawa sessions to see issues the model faced while installing and enabling sshd.</q> The first address I gave was wrong by one digit. The second worked.

Nawa had left some surprises behind: a second SSH server on port 2222, a "Match LAN" block, a hole in the firewall. Claude backed them up and cleaned them out. A week later bunix ships its own OpenSSH, off by default, with a firewall in front of it.

{{< stamp >}}Wednesday, 4 November · all day{{< /stamp >}}

### Too big to boot, and a flicker

Overnight the ISO had grown to 777 MB, and it could no longer boot on a machine with 1 GB of memory, because the live system copied everything into RAM. The redesign mounts a verified `/usr` straight from the disc. That brought the ISO to 640 MB, with 78 MB of memory in use. On the testbed, the boot loader went from 13.8 seconds to 4.

Phase three started at 07:00. <q class="user">from the start prepare integrating libghostty instead of foot.</q> Then a decision that shaped the release: the whole desktop would live in its own optional extension. <q class="user">during installation the user can decide whether to have desktop or just console.</q>

LLVM came next, built for Mesa. <q class="user">why does it take all that to build LLVM? what are we actually building?</q> pro13 started swapping, with the CI machines running beside it. <q class="user">shut down the CI VMs on pro13. bunix won't comeback until 3 hours.</q>

At 15:44 the testbed went dark for a different reason. <q class="user">my daughter shut testbed down, she wants to use Windows. I'll put it back after she sleeps.</q> It came back at 17:27. No build is worth more than that.

Then I sent a video. <q class="user">watch this video and notice the flicker in the boot splash.</q> The cause took a while to find. The firmware drives the panel's clock with spread spectrum; the Intel driver doesn't. So the handover from firmware to driver can't be seamless, and the screen blanks for one to three seconds. No kernel option fixed it. <q class="user">we'll deal with it later.</q> Some bugs belong upstream, and knowing which ones is part of the job.

{{< part id="part-3" kicker="Part three" title="A desktop" >}}Thursday 5 November to Friday 6 November{{< /part >}}

{{< stamp >}}Thursday, 5 November · 04:35 · the testbed{{< /stamp >}}

### Key was rejected by service

0.1.10 brought the first desktop. After its reboot, the testbed disappeared from the network. I set a root password and logged in at the console. <q class="user">networkctl only shows the loopback interface!</q> Then: <q class="user">modprobe fails : could not insert r8169: Key was rejected by service.</q>

This was a problem with how builds were shared, not with the code. To save time we had been copying build results between pro13 and the testbed. Each kernel build signs its modules with a fresh random key. The copy had paired pro13's kernel with the testbed's modules. Every module was rejected, so there was no network and no graphics.

Back on 0.1.9, I worried about the obvious thing. <q class="user">but now when we update to 1.11 we will lose 1.9 so we can't recover if 11 fails!</q> We wouldn't: the update always overwrites the slot you're not running, which held the broken 0.1.10. From then on, the build itself refuses to produce a boot image whose modules don't match its kernel.

{{< lesson >}}Sharing work between machines is only safe when the thing you share means the same thing everywhere. Remember this one; it comes back in chapter 17.{{< /lesson >}}

{{< stamp >}}Thursday · 05:29 – 10:34{{< /stamp >}}

### The definition of done

With Hyprland running, I drew the line for the first real release. <q class="user">no rush for publishing a version, I prefer I publish when we have a usable system with login and terminal in Hyprland as 0.2 version.</q>

At 08:07: <q class="user">ghostty ran correctly, version shows 1.4.</q> At 10:34 the look was settled: <q class="user">for the design system we should be using Banan Desgin System with dark and light versions.</q> Light, plus a Snazzy dark, with a clay accent and the Geist typeface. These pages use it too.

{{< lesson >}}Say what "usable" means before you ship. A version number promises something, so decide what it promises first.{{< /lesson >}}

{{< stamp >}}Thursday · 08:29 → 21:18{{< /stamp >}}

### Silence, three times

I asked Nawa to install Chromium from Google, mostly to see whether it would. <q class="user">Sound is not working tho.</q> Then, an hour later: <q class="user">I don't here any voice!</q> The first cause was the simplest: the sound card's master channel was muted, and nothing restored it at boot. After a reboot: <q class="user">sound worked without running anything.</q>

In the evening it came back, this time for my own account rather than root. `amixer` said <q>cannot find card '0'</q>. My user wasn't allowed to touch the device, because systemd needs a small library to grant that access. I pushed back. <q class="user">base should be minimum and universal, is this library is going to be used with sounds only, then I see it as a Desktop concern.</q> It isn't only about sound, though; it's how every device is handed to whoever is sitting at the machine. It went into base. <q class="user">yep, sound works now.</q>

The third time, PipeWire was running, the test tone played front left and front right, and Chromium was still silent. <q class="user">wait, does chromium require libpulse? in another session I started building our own chromium based browser.</q> Google's build carried its own copy of the sound library, which knew nothing about PipeWire. Moving it aside fixed it. <q class="user">yep, working now.</q>

Three silences in one day, with three different causes. A fourth was still to come.

{{< stamp >}}Thursday · 23:00{{< /stamp >}}

### “Aweful”

By night we had users, PAM, yescrypt passwords (<q class="user">yescrypt it is then, make it work, no mistake :)</q>) and a login screen of our own, buna-login. I recorded a video of the boot on my phone and sent it. <q class="user">functionalityy is working, but UI and UX is aweful!</q>

It was. Boot logs scrolled before the login screen appeared. A yellow banner sat across the top of Hyprland. The login screen used a whole desktop compositor just to show one password field.

The fixes came over the next morning. The splash now stays up until the login screen is ready. The logs went to the journal. The banner went. <q class="user">can't we use a minimal compositor for login screen instead of loading hyprland?</q> Cage for now, and later one of our own: <q class="user">I'm thinking of writing our own in Rust by linking aquamarine instead of Smithy, an important feature it should is on screen keyboard for touch only machines and when there is only one user we show password field only.</q> The on-screen keyboard got an Arabic layout, and the kernel got uinput so a fake touchscreen in QEMU could test it.

{{< lesson >}}Working and good are two separate finish lines. The second one is measured by watching someone use it.{{< /lesson >}}

{{< stamp >}}Friday, 6 November · 14:14 · GitHub{{< /stamp >}}

### 0.2.0

After thirty-two internal versions, a fresh install reached the login screen, then Hyprland, then Ghostty, and the installer asked for the computer's name. <q class="user">publish.</q> bunix 0.2.0 went out at 14:54. Then: <q class="user">ok, start next step, but don't touch the testbed until I tell you. I will install 0.2 on it from scratch.</q>

An hour later, a photo from a USB install. The installer couldn't mount the USB stick it had booted from: <q>Can't open blockdev</q>. And `iwctl`, the Wi-Fi tool, crashed at its own prompt. That one came from a bug in GNU readline, with patches waiting upstream.

I made two more calls that afternoon. Hold 0.2.1 for the browser, which the other session was building. And new keys: <q class="user">configure Hyprland to open Ghostty with SUPER+RETURN and the browser with SUPER+SHIFT+RETURN, while quitting Hyprland (SUPER+M) causes a logout.</q>

{{< part id="part-4" kicker="Part four" title="Flow" >}}Friday night 6 November to Saturday 7 November{{< /part >}}

{{< stamp >}}Friday · 18:16 – 23:51{{< /stamp >}}

### Don't wait for the browser

The readline patches were small. Their effect wasn't. In BuildStream, changing one package rebuilds everything above it, and above readline sits Python, and above Python sits almost everything. Ninety-eight elements went back into the queue, and eventually Chromium, which takes about nine hours.

I let go of holding the release. <q class="user">browser is going to take 9 hrs, don't wait for it.</q> And at 20:08: <q class="user">ok, publish once tests pass.</q> USB install, Wi-Fi, an update from 0.2.0. At 23:51, 0.2.1 shipped, without the browser and with the new keys.

While it uploaded, I asked something unrelated. <q class="user">how does macOS handle rollbacks? they don't have A/B partitions, no?</q> They don't: macOS boots from a sealed, signed snapshot, and rolling back means booting the previous one. <q class="user">I like the macOS design better tbh, I might start do some master's research on a similar design for linux.</q> A reading list went into a document for later.

{{< lesson >}}Ship what's ready. A release that waits for its slowest part ships nothing.{{< /lesson >}}

{{< stamp >}}Friday · 21:16 → Saturday 01:00 · pro13's USB disk{{< /stamp >}}

### The constraint

The ninety-eight rebuilds stuck with me. <q class="user">why id we need to build everything? can't we distribute the build over to tetbed and fly?</q> Then: <q class="user">we should have a dependency graph and distribute.</q>

This was the turning point of the week. Every change upstream of Chromium has to pass through Chromium, and Chromium takes most of a day each time. Every machine that built it separately repeated the same day. Nothing else we could speed up would matter as much as not doing that work twice.

<q class="user">the goal is to have declarative, reproduceable, distributed build system.</q> That was the Sunday sentence again, grown up. We wrote down the design goals, with Bazel, Gradle's build cache and Nix as prior work. Then we took the two small steps that could be done that night:

First, `tools/build-graph.py`, which shows what a change will rebuild and the longest chain that no number of machines can shorten. Second, one cache shared by every builder. <q class="user">lets make sure the cache is on the USB attached disk on pro13.</q> Buildbarn, 140 GiB set aside on that disk, with 91 GB left over for the CI machines, measured rather than guessed.

{{< lesson >}}Find the constraint first. Any improvement that doesn't touch Chromium's nine hours is an illusion.{{< /lesson >}}

{{< stamp >}}Saturday, 7 November · 00:00 – 06:00 · Frankfurt{{< /stamp >}}

### The night shift

I resized the Fly machine to 16 cores and 48 GB myself, since spending money is my call, and the browser build started there, pulling 33 packages from pro13 instead of compiling them. Then it hit problems. The machine had no `patch` program. Then an SSH session dropped and took the build down with it. After that, long builds ran detached, logging to a file and not depending on the connection that started them.

Meanwhile I reinstalled the testbed from scratch, as a normal person would: <q class="user">there are some quirks, but it working in general.</q> <q class="user">note the issues in a file so we can fix them later.</q> `docs/known-issues.md` was born.

I sketched a package manager, `buna install sqlite`, and put it on the shelf. <q class="user">no, later. we can work on gui installer now.</q> The installer would need a screen to run on, so the night ended with buna-kiosk, our own small compositor on aquamarine. It painted every screen the login screen's warm off-white and survived a switch to a text console and back.

{{< stamp >}}Saturday · 00:14 – 07:30{{< /stamp >}}

### Trust, and the watcher that watched itself

Every step ends with a review, by my standing rule: two reviewers in parallel, one hunting bugs, one asking whether a stranger could read the code. This round found a compositor that would spin a CPU core forever if a screen refused to turn on, and cache tunnels that killed each other when two builds ran at once. It also raised a trust question. Releases are signed from pro13's cache, and anyone logged in to the Fly machine could write to it.

That's the r8169 lesson in a new form: what you share has to be what you think it is. <q class="user">ok, enforce pull-only and refuse to sync onto a running build.</q> The cache gained two ports that only allow reads, and the build scripts now refuse to copy over a build in progress. Both were proved by watching them fail on purpose.

Chromium had finished at 05:48, after five hours and thirty-two minutes, and nobody was told. The background check meant to report it searched for any running `bin/bst`, found its own command line, and reported "still running" forever. Monitoring is code too, and when it's wrong it doesn't fail; it just keeps reporting things are fine.

{{< stamp >}}Saturday · 07:30 – 08:18{{< /stamp >}}

### Five seconds

Fly pushed its work into the shared cache: 86 packages. Then pro13 built the desktop and the ISO. Its cache keys matched Fly's exactly. pro13 asked the cache for Chromium and had it in five seconds. The desktop extension and the ISO took three minutes.

That was the moment the week had been building towards, though I hadn't known it on Sunday. Work done once, stored where every machine can find it, is never done again.

<q class="user">first push main, and publish a release, I want to update the testbed and see the browser.</q> The exact ISO was installed in QEMU and tested before anything went public, then twelve files went up one by one, each checked by size. bunix 0.2.2 shipped at 08:18 with Buna Browser.

{{< stamp >}}Saturday · 08:25 · the testbed{{< /stamp >}}

### Silence, the fourth time

<q class="user">rebooted, sound is not working in the browser!</q>

Everything looked healthy: PipeWire running, speakers present, `aplay` working. But the browser had never opened a stream. Claude rebuilt the failure in QEMU with a virtual sound card that recorded to a file. The first launch failed. A second, a few seconds later, played.

Then: <q class="user">what did you do on the testbed? sound is working now! I haven't restarted anything.</q> Nothing; every command there had been read-only. The timestamps explained it. PipeWire only started when something first asked for sound. The browser was that first program. PipeWire came up a moment later, the speakers a moment after that, and by then the browser had given up. Any later try worked.

The obvious fix was to start PipeWire with the session, in PipeWire's own package. But Chromium builds against that package, and touching it would have meant another five and a half hours. So the line went into WirePlumber's package instead, which nothing heavy depends on. Five small steps rebuilt. On a fresh install the browser's first launch played a tone, and the recording caught it.

{{< lesson >}}Keep the constraint in mind even when you're fixing something else. The same one-line fix cost three minutes in one place and five and a half hours in another.{{< /lesson >}}

{{< stamp >}}Saturday · 08:45 – 09:00{{< /stamp >}}

### Closing for the week

<q class="user">delete the fly build machine, enough work on bunix this week.</q> The machine in Frankfurt was destroyed. Nothing it built was lost; it was all in the shared cache on pro13's USB disk.

I asked for this story. The first draft started on Friday night. <q class="user">don't you have the full history of this claude code session? it's been running for about 5 days now!</q>

It did, and it didn't. A long conversation gets summarized to make room, twelve times this week, and what Claude remembered was the summary. The full record was still on disk, line by line. So the story was rebuilt from the log, the same way we'd been building everything else: from what was written down, not from memory.

* * *

It's tempting to measure the week by what shipped: an operating system compiled from source, nine releases, a desktop, a browser. What I'll remember is the flow of work. On Sunday the problem was a shell that Rosetta couldn't run. By Saturday it was making sure the same nine hours of work never happens twice. The week taught three things:

**Flow.** Find the constraint and stop repeating work there. The shared cache, the build graph, and putting a fix where it doesn't trigger a rebuild.

**Feedback.** Shorten the distance between a change and its truth. QEMU with recorded sound and typed keys, real hardware on a dual-boot laptop, phone photos of a boot screen, two reviewers after every step.

**Learning.** Write it down. A known-issues file, design documents, a reading list for a master's thesis, and a log good enough to rebuild a week from.

#### The ledger {#ledger}

<div class="tablewrap">
  <table class="ledger">
  <tr><td>Days, from minisys ISO to Buna Browser</td><td>8</td></tr>
  <tr><td>Releases published on GitHub</td><td>9</td></tr>
  <tr><td>Internal versions before 0.2.0</td><td>0.1.0 – 0.1.32</td></tr>
  <tr><td>Commits on main</td><td>43</td></tr>
  <tr><td>My messages</td><td>271</td></tr>
  <tr><td>Times the conversation was summarized</td><td>12</td></tr>
  <tr><td>Speed-up from moving off Rosetta to pro13</td><td>about 4×</td></tr>
  <tr><td>First bunix ISO</td><td>61 MB</td></tr>
  <tr><td>ISO that would not boot in 1 GB of RAM</td><td>777 MB</td></tr>
  <tr><td>QEMU boot, kernel plus userspace</td><td>1.8 s</td></tr>
  <tr><td>Elements rebuilt by six readline patches</td><td>98</td></tr>
  <tr><td>Chromium on Fly, 16 cores</td><td>5 h 32 min</td></tr>
  <tr><td>The same Chromium, pulled by pro13</td><td>5 s</td></tr>
  <tr><td>Shared cache on pro13's USB disk</td><td>140 GiB</td></tr>
  <tr><td>Times the sound stopped working</td><td>4</td></tr>
  <tr><td>Times the testbed was handed back for Windows</td><td>1</td></tr>
  <tr><td>Fly build machines still running</td><td>0</td></tr>
  </table>
</div>

<footer>

Claude wrote this from the full session log of 31 October – 7 November 2026, in the style of a DevOps business novel. The quotes are my messages as I typed them, typos included. bunix lives at [github.com/bunasys/bunix](https://github.com/bunasys/bunix).

</footer>
