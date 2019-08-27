---
layout: post
title: Rust Raspberry Pi 4 cross-compilation from Windows
---

In this post I will go through how to install and configure a Rust cross-compilation environment from Windows to the Raspberry Pi 4 running Raspbian. I am assuming that Rust is installed in the Windows system following [these instructions](https://www.rust-lang.org/tools/install) and that a Raspberry Pi 4 machine is running with the name *raspberrypi* and accessible by ssh.

## Installing the C/C++ toolchain

The first step is to install a C/C++ toolchain for the Raspberry Pi. I typically use the one provided by [SysGCC](https://gnutoolchains.com/raspberry/) but I am sure there are others available. To test that the cross-compiler is working you can write a simple *main.c* file such as

```c
#include <stdio.h>

int main()
{
    printf("Hello World\n");
    return 0;
}
```

Then run the following commands on a PowerShell to compile and run the executable on the remote Raspberry Pi. 

```console
    arm-linux-gnueabihf-gcc-8.exe .\main.c -o mytest
    scp .\mytest pi@raspberrypi:/home/pi
    ssh pi@raspberrypi chmod +x /home/pi/mytest
    ssh pi@raspberrypi /home/pi/mytest
```

If you see the **Hello World** output on your PowerShell terminal then the cross-compilation went well and everything is working.

## Prepare the Rust environment

Now it is time to get the Rust environment ready for cross-compilation. First start by installing the target using the PowerShell.

```console
rustup target add armv7-unknown-linux-gnueabihf
```

Then edit (or create) the file *C:\\Users\\<username>\\.cargo\\config* by adding the line

```
[target.armv7-unknown-linux-gnueabihf]
linker = "arm-linux-gnueabihf-gcc-8.exe"
```

This line configures the linker to be used for creating the executable. If this is not done, a *error: linker `cc` not found* message shows up when trying to compile an executable for that target.

## Cross-compiling Rust hello world

To create a Rust "Hello World" program, simply make a new folder (in this case I call it *rs-hello*) and let cargo do the rest of the work for you.

```console
cargo init
cargo build --target=armv7-unknown-linux-gnueabihf
```

Now there is an executable that can be copied and executed on the remote Raspberry Pi.

```console
scp .\target\armv7-unknown-linux-gnueabihf\debug\rs-hello pi@raspberrypi:/home/pi
ssh pi@raspberrypi chmod +x /home/pi/rs-hello
ssh pi@raspberrypi /home/pi/rs-hello
```

If you see the **Hello, world!** output on your PowerShell terminal then the cross-compilation went well and you can now create Rust programs to run on your Raspberry Pi.