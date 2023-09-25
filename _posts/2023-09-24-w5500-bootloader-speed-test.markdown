---
layout: post
title:  "Comparing TCP send speeds of Wiznet W5500 bootloaders"
date:   2023-09-24 20:00:00 -0400
categories: jekyll update
---

In my [previous post]({% post_url 2023-09-21-w5500-intro %}), we got up and running with the W5500, and touched lightly on the various bootloader options for the board.  I had some issues on the client side of the [Wiznet releases](https://github.com/Wiznet/RP2040-HAT-MicroPython/releases), and in the article suggested just sticking to the [Official Micropython builds](https://micropython.org/download/W5500_EVB_PICO/), as they didn't seem to suffer from the same issues.

Turns out, I jumped the gun a bit.

While writing the article and testing things out, I never went back and reperformed the W5500 to W5500 client/server test after applying the official Wiznet bootloader.  When I sat down to start working on my enhancements to my examples, I found that I was getting really poor speeds when sending from client W5500, regardless of whether it was to the server running on the W5500, or on my PC.  When performing the last tests in that article, I just assumed it was the LED value toggles or the print statements slowing things down, but even once I pulled that all out, no dice.  For a sanity check, I retried the Wiznet releases, both the `1.0.5 stable` build as well as the `2.0.0 prerelease`.  It turns out the `1.0.5` build was much quicker than the Micropython build (neither the stable or nightly builds made a difference, `v1.20.0 (2023-04-26) .uf2` and `v1.20.0-489-ga3862e726 (2023-09-20) .uf2` respectively as of this writing), and the `2.0.0` was *much* faster even than the `1.0.5` build.

Phooey.

This was pretty unfortunate, as obviously we're looking for reliability, as well as speed here, and the speeds I was seeing were nowhere near acceptable, we'll get to some specifics later on.  So, what to do?  Time for looking around for *another* option.

## Circuitpython

When I got my first batch of Pico Pis, I had down a little bit of research prior on the ecosystems around them.  I plan on utilizing them with a combination of off the shelf parts such as OLED screens, LCDs, 7-segment displays, as well as discrete components like switches, LEDs, as well as my own one off hardware utilizing, SPI, I2C, etc.  Basically, the full range of hobbyist electronics "stuff".  During that initial googling around for "micropython + \[X\]" vs "circuitpython + \[X\]", the vibe I got was that there was more available on the micropython side.  I also really liked the CLI support with `rshell`, so I started down that path.  

After running into the issues described above, I knew I had to take another look at circuitpython, and when I found the github repository for the W5500 support I realized I might have jumped the gun on passing on circuitpython originally.  The project seems to be much more mature, as well as more recently and consistenly updated than the micropython equivalent.  But, the million dollar question remained, how does it compare in the stability and speed test.  Let's take a look.

## The Testing Rig

## Wrapping Up

I hope this has served as a solid foundation to your own hacking with the W5500.  As I said, I've got a project I'm in the prototyping stages of that currently uses regular Picos communicating over USB that I'm looking forward to converting over to ethernet. While these proof-of-concepts I've provided can get you up and running, they're nowhere near production ready.  If you're looking for next steps to expand on this, I'd suggest hardening the connection standup and come down, such that the client will loop attempting to connect to the server, and the server can handle clients coming and going.  Another good exercise would be running the server on the W5500, and creating a script to run on your PC that talks to it, reversing the example we provided above.

If you come across any errors, or have suggestions for how better to present anything, please don't hesitate to reach out to me at [ghp@stephanj.com](mailto:ghp@stephanj.com).  Happy hacking!