---
layout: post
title: Debugging embedded Rust programs using VS Code
---

In our previous posts, we wrote about generating Rust code for an STM32F7 processor. This was enough to get a simple LED blinking program working but for creating more complicated software we need to be able to develop, compile, deploy and debug from an IDE. We will use the same source code and many of the same tools as in the previous post so make sure to have a look if you want the complete picture. The tools you need to have installed before starting are [VS Code](https://code.visualstudio.com/) with the Rust(rls) extension, a cross-compilation toolchain and GDB debugger (we are using the one provided by [ARM](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads)) and OpenOCD (for example from [here](https://gnutoolchains.com/arm-eabi/openocd/)) as the remote debugging server.

## Building from VS Code

The first step to get our setup running is to generate a binary file using VS Code. To achieve this we need to create two tasks, one to compile the Rust code and the second to convert the resulting ELF file to binary. This can be achieved by adding a *tasks.json* file with the following definitions

```json
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558 
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Cargo build",
            "type": "shell",
            "command": "cargo",
            "args": ["build", "--release"],
            "problemMatcher": [
                "$rustc"
            ],
            "group": "build"
        },
        {
            "label": "Build binary",
            "type": "shell",
            "command": "arm-none-eabi-objcopy",
            "args": [
                "--output-target", "binary",
                "./target/thumbv7em-none-eabi/release/blinky",
                "./target/thumbv7em-none-eabi/release/blinky.bin"],
            "problemMatcher": [
                "$rustc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "dependsOn": "Cargo build"
        }
    ]
}
```

To make sure that the build command creates the correct target and that the debug symbols are kept, the *cargo/config* file need to have the following contents:

```
[target.thumbv7em-none-eabi]
rustflags = ["-C", "link-arg=-Tlink.x", "-g"]

[build]
target = "thumbv7em-none-eabi"
```

Note that, compared to the previous post, we added the *-g* flag. This ensures that the release compilation keeps the debug symbols to allow using breakpoints and stepping through the source code.

Running the **Build binary** task executes the two steps needed to get a binary file.

## Debugging on VS Code

To debug on VS Code we are using the [Cortex-Debug extension](https://marcelball.ca/projects/cortex-debug/), which does most of the hard work of coordinating OpenOCD and remote GDB connections. We use the following launch configuration on VS Code:

```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug Blinky",
            "request": "launch",
            "type": "cortex-debug",
            "cwd": "${workspaceRoot}",
            "executable": "${workspaceFolder}/target/thumbv7em-none-eabi/release/blinky",
            "svdFile": "${workspaceFolder}/STM32F7x6.svd",
            "servertype": "openocd",
            "configFiles": ["st_nucleo_f7.cfg"],
            "preLaunchTask": "Build binary",
            "preLaunchCommands": [
                "monitor init",
                "monitor reset init",
                "monitor halt",
                "monitor flash write_image erase ./target/thumbv7em-none-eabi/release/blinky.bin 0x08000000"
            ],
            "postLaunchCommands": ["continue"] // Don't stop before at the first line
        }
    ]
}
```

There are a few adjustments compared to the default launch configuration offered by the extension.

- The `"svdFile": "${workspaceFolder}/STM32F7x6.svd"` command simply adds the possibility of seeing the processor registers during debugging.
- Typically, we like to have the code rebuilt before launching, which is done by `"preLaunchTask": "Build binary"`. The resulting binary is also flashed using OpenOCD by the command block:

```json
"preLaunchCommands": [
                "monitor init",
                "monitor reset init",
                "monitor halt",
                "monitor flash write_image erase ./target/thumbv7em-none-eabi/release/blinky.bin 0x08000000"]
```

These are commands from OpenOCD so refer to the documentation for more details

- The Cortex-Debug extension stops the execution either at the first function or at the main (configurable by `"runToMain"`). To avoid this we add the `"postLaunchCommands": ["continue"]` command to keep running without interruptions. All of these additions could be removed or adjusted depending on your preferences.

Now we can deploy, run and debug our programs with a single button click.

![VS Code STM32F7 Debug example]({{ site.baseurl }}\images\vs_code_stm32f7_debug_example.png)



If you are interested, you can get all the files for this project on the [github page](https://github.com/s2e-systems/stm32f7-blinky/blob/master/src/main.rs).