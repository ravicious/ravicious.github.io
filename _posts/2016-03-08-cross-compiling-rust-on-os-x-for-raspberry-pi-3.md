---
layout: post
title: Cross compiling Rust on OS X for&nbsp;Raspberry Pi 3
---
I&nbsp;managed to get my hands on Raspberry Pi 3B. I&nbsp;found it to be a&nbsp;great opportunity to mess around with Rust. But some problems appeared right away: how do I&nbsp;run my Rust programs on Raspi? Can I&nbsp;compile them on my Mac or&nbsp;should I&nbsp;compile them on the device?

I&nbsp;didn't have much experience with compiling stuff for other platforms, but turns out it's not that dramatic – at least for the simple ones I&nbsp;wrote so far.

<!--more-->

## The thing that's complicated on OS X

The process we're interested in is called [cross compilation](https://en.wikipedia.org/wiki/Cross_compiler). To cross compile a&nbsp;program for other platform, we need a&nbsp;suitable toolchain.[^1] One can find many tutorials on how to cross compile Rust code for Raspberry Pi, but most of them are for Linux users. We could trying building a&nbsp;toolchain by ourselves, though doing that on OS X is a&nbsp;total pain. The amount of outdated information on the Web and the number of hurdles we'd have to go over renders Macs unusable for that.

If you don't believe me, check out the outdated docs on [how to get crosstool-ng to work on OS X](http://crosstool-ng.org/hg/crosstool-ng/file/715b711da3ab/docs/MacOS-X.txt). The main issues with OS X are [the incompatibility of FreeBSD-based command line tools](http://unix.stackexchange.com/a/82248/45904) with their GNU counterparts and [the case insensitivity of HFS+](https://en.wikipedia.org/wiki/HFS_Plus#Limitations).

## The solution

Instead, let's use information gathered in [Erik's rust-on-raspberry-pi repo](https://github.com/Ogeon/rust-on-raspberry-pi). While building the cross compiler by hand won't work for us, [schnupperboy](https://github.com/schnupperboy) dockerized the whole process which makes it a&nbsp;breeze.

I&nbsp;was skeptical at first, as I&nbsp;didn't use Docker before and thought it's an overkill. Turns out it's pretty handy for this use case and saves us a&nbsp;lot of hassle. After installing [Docker Toolbox](https://www.docker.com/products/docker-toolbox) and going through its [Getting Started Guide](http://docs.docker.com/mac/started/), we should be able to follow instructions from rust-on-raspberry-pi's DOCKER.md without any issues. Keep in mind that Docker commands have to be run from the Docker Quickstart Terminal session.

I&nbsp;[ran into problems with permissions in Docker](https://github.com/Ogeon/rust-on-raspberry-pi/issues/10), but after some digging I&nbsp;found the post called ["Web development with Docker on Mac OS X"](https://medium.com/@brentkearney/docker-on-mac-os-x-9793ac024e94). The user created inside the Docker image needs to have `uid` set to `1000` to access files from the host filesystem. The PR which fixes this has been already merged into the rust-on-raspberry-pi master branch.

It took me a&nbsp;couple of days, but cross compiling finally works on my machine and the compiled programs seem to run well on Raspi. I&nbsp;got reminded once again that with such dynamic technologies, the blogposts written two years ago could as well not exist. It's the reason I&nbsp;wrote this one and I&nbsp;hope it'll serve people for another few months. 🎉



Shout out to [Arek](http://twitter.com/aflinik) and [all the amazing people at Estimote](http://careers.estimote.com/) thanks to whom I&nbsp;was able to experiment with Raspberry Pi!



```shell
$ docker run -v /Users/rav/Projects/rust/guessing_game:/home/cross/project rust-1.7.0-raspi build --release
*** Extracting target dependencies ***

*** Cross compiling project ***
    Updating registry `https://github.com/rust-lang/crates.io-index`
 Downloading rand v0.3.14
 Downloading libc v0.2.7
   Compiling libc v0.2.7
   Compiling rand v0.3.14
   Compiling guessing_game v0.1.0 (file:///home/cross/project)
$ scp target/arm-unknown-linux-gnueabihf/release/guessing_game pi@raspberrypi.local: 
guessing_game                           100%  525KB 525.0KB/s   00:00    
$ ssh pi@raspberrypi.local
pi@raspberrypi:~ $ ./guessing_game 
Guess the number!
Please input your guess.
```

[^1]: As [Mats Petersson said on SO](http://stackoverflow.com/a/22756355/742872), a toolchain is a set of tools needed to produce executables for the target platform, like a compiler, a linker, shared libraries and so on.
