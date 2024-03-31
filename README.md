# WSL for Embedded Development

## What is WSL?

[Windows Subsystem for Linux](https://learn.microsoft.com/en-us/windows/wsl/about) is a lightweight VM that allows you to run a Linux environment directly and easily on Windows without any other software such as VMWare or VirtualBox. It runs with a true Linux kernel (written by Microsoft and interacing with the VM) and a true ext4 filesystem (which exists as a vhdx file in the AppData folder of your C drive).

> __Note:__ This article is about WSL2, not WSL1.

## WSL Alternatives

### Why not Mac OS?

> __TODO:__ Consolidate and shrink this section.

Not too long ago, it used to be the case that if you were an embedded software developer you worked on Windows. In fact, you probably [didn't have a choice](https://interrupt.memfault.com/blog/the-best-and-worst-mcu-sdks#dont-choose-my-laptop-os-for-me). But, more and more, people are looking for a POSIX-compliant environment that can handle Bash scripting and CLI-based tooling. Apple has positioned themselves well to pick up this market share with their switch to the BSD-based [Darwin](https://en.wikipedia.org/wiki/Darwin_(operating_system).

Mac OS may be POSIX-compliant, but Linux it is not and Apple has their own way of doing things. I've seen a number of ways that this creates issues for my coworkers. Following is a short list of the issues I've observed.

- The switch to M1 silicon was a major shake-up. It has taken a while for compilers and other tools to catch up. It has created issues with running x86-based Docker images locally (now they have to be multi-platform or rely on Rosetta). And some apps are simply lost in the annals of time (something that is a bigger deal for embedded development than it is for other software industries).
- Apple clang masquerading as gcc. Not only does it define the `__GNUC__` intrinsic, it calls itself `gcc` on the command line. For the most part, this goes unnoticed, but you do have to watch out around compiler-detection conditional compilation flags and around subtle variations in command-line arguments. I've definitely seen this trip up some projects.
- Apple switched the default shell to zsh starting with Catalina. This is because the license for bash changed to GPLv3. A lot of people like zsh, but I have seen it create some sharp edges. Bash and zsh are not the same thing and sometimes a shell script expects to be running in bash (and not the outdated 3.2 version of bash either).
- Many command-line utilities are different on the Mac. Apple likes to make their own versions of everything with slighltly different command-line options. It's another sticking point for the cross-compatibility of shell scripts between Mac and Linux.
- APFS has case-insensitive behavior. Unlike the ext4 filesystem, it cannot allow two files of the same name, differing only in case, to exist in the same directory. This creates problems in certain scenarios, particularly with Yocto.
- Although the [Homebrew](https://brew.sh/) logo appears to depict beer, it is actually cider, as indicated by the Apple. Linux is free as in beer. Beer is better than cider. Therefore, Linux is better than Mac OS (and by extension, so is WSL).

### Why not Docker?

Memfault has already [touched on this](https://interrupt.memfault.com/blog/conda-developer-environments#common-developer-environment-alternatives), so I won't say too much more here. What I will say is that the "How do I connect to hardware?" question is far more straightforward with WSL than it is with anything like Dev Containers or GitPod.

> __Note:__ I am aware of the ability of cloud based IDEs (such as Keil Studio and Arduino Create) to connect with hardware. This is done via an agent installed on the user's machine that interacts with the browser. This article assumes no such special software in setting up a workflow.

## WSL Benefits

The primary benefit is knowing that you're operating in a true Linux environment including a Linux kernel, an ext4 filesystem, and GNU-compliant Bash shell and command-line utilities. Considering that anything running in CI is likely to be a Linux-based Docker image, this is just a cleaner way to go. I have caught many issues where something worked fine on Mac but not on Linux due to a WSL-based workflow.

The other benefit is how much faster builds run in WSL than in native Windows. I have experienced much faster build times in WSL as compared to Windows. For a typical bare-metal project with a CMake/Ninja based build, a pristine build may be under 10 seconds.

> __TODO:__ Add graph of example project with build-time results for different environments.

## Getting Started with WSL

I won't cover getting started with WSL in this article, but Medium has a good article [here](https://medium.com/@awlucattini_60867/getting-started-with-wsl2-c11826654776). The only difference to it I would make is to say that Ubuntu 20.04 is outdated. Go with Ubuntu 22.04 instead.

## WSL Integrations

Let's move on to the main question: How to effectively and seamlessly integrate WSL into the development workflow. As I stated earlier, I'm not going to give a single prescription. I will simply list all the ways that WSL facilitates workflow integration.

### Cross-mounting of Filesystems

If you are wondering how to access files, the good news is that Microsoft has you covered in both directions.

- All Windows letter drives are mounted in the `/mnt` directory of the WSL instance. This means that all files from your NTFS partitions are available within WSL.
- All WSL instances are accessible in File Explorer as a network share beginning with `\\wsl.localhost`. Explorer navigating to a WSL filesystem easy with a top-level "Linux" category in the folders pane.

![Linux-shares](images/Linux-shares.png)

### Mapping a Drive Letter

Being able to access files within the WSL instance through a network share is powerful, but it does have some limitations. Some apps will work well with that (e.g. Balena Etcher) and others will not. To get around this, it's necessary to map the WSL instance to a drive letter. Right click on the top-level folder (e.g. "Ubuntu-22.04") and select "Map network drive...". For the drive letter, I typically choose "Y:" or "Z:". Now you should have no problem opening IDEs, editors, etc. natively in Windows to the contents of a WSL instance. This also makes it easy to navigate to files in the WSL instance from a Windows-side command shell. This is a useful trick for running flash and debug scripts from Windows.

### Network Adapter

Windows creates a Hyper-V Virtual Ethernet Adapter for WSL. Primarily this allows access to the Internet from WSL, but it also allows Windows-side and WSL-side applications to talk to each other via TCP/IP. This is similar to how containerized applications can communicate with the outside world via [exposed ports]9https://docs.docker.com/reference/cli/docker/container/run/#publish).

To take advantage of this, you will want to know the IP address assigned to the WSL adapter by Windows and you will want to know this inside the WSL instance. Add the following line to your `.bashrc` to assign this address to an environment variable.

```bash
export HOST_IP=`ip route|awk '/^default/{print $3}'`
```

### Forwarding of Windows PATH Variable

Type `echo $PATH` and you will see the contents of `echo %PATH%` as executed in `cmd.exe` following all the WSL-native paths. The only difference is that the paths are modified to work from WSL. (`C:\` changed to `/mnt/c/`, all `;` separators changed to `:` etc.). Type the name of any Windows executable visible in the Windows path and it will launch from the WSL Bash shell. For example, type `explorer.exe .` and a new instance of File Explorer will open showing the contents of the current WSL directory. The WSL Linux kernel automatically forwards any invocations of a `.exe` file to Windows.

### WSL.exe

The WSL.exe utility can be used for managing WSL instances, but it can also be used for invoking scripts within a WSL instance directly from Windows. This can be used to invoke a build script, or any other script, from an IDE residing natively in Windows.

Example: Let's say I have my `stm-blinky` project cloned in WSL in a projects subdirectory of my user home directory. That project is managed using xPack and uses the `xpm run` command to invoke the build process. Invoking the build process from Windows would look as follows:

```cmd
wsl.exe --cd /home/AFontaine79/Projects/stm-blinky xpm run build --config Debug
```

> __Note:__ This example assumes that Ubuntu-22.04 is my default WSL instance.

### VS Code

VS Code makes integration with WSL easy thanks to vscode server. This is the same remote connection technology used for connecting to dev containers or cloud-based dev environments. Simply navigate to the base directory of the repo you want to develop on and type `code .`. VS Code will open automatically open in Windows and establish the remote connection to your WSL instance. The terminal and all the files will operate natively in the WSL instance.

#### Debugging with VS Code

Any tasks or launch configurations will operate in the WSL instance and therefore not see your debug hardware. My primary recommendation is to use WSL USB GUI, discussed below, to map your JLink or similar hardware into WSL and work from there. However, it is possible to launch the JLink GDB Server on the Windows side and have the `gdb` client in WSL connect to it.



## Deleted content

### Why I Stuck with Windows

Simply put, I'm a Windows user plain and simple. When it comes to UI, I like the way Microsoft does things better than Apple. And UI philosophy has a huge impact on developer workflow. I used a MacBook throughout the 2010s in my personal life and I'm just not going back.

I have stuck with Windows in my professional life despite the ongoing shift to POSIX-compliant systems, Bash scripting, and the increasing reliance on Unix-style command line tools. I understand how awkward and backwards Windows looks against this backdrop. Part of me thought that the day would come that I would have to give up on Windows, but somehow it never has.

To be honest, the cross-platform support out there is really good. Git Bash gives an easy deafult Bash shell for Windows. Apps like PowerToys and Terminal give a really good experience. And overall, for one that knows how to use this platform, the problems are negligable to non-existent.

Most of the problems I see other developers having with Windows come down to one of two things. One, they are just not familiar with the platform. Or two, there is some issue with the project setup. Perhaps it's not set up properly for cross-platform development. Or perhaps it's not following best practices and just happens to work on a Mac. Number two should be a concern for any project, especially considering that CI jobs run in a Linux environment.
