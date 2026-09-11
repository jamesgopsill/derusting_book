# Prerequisites

This section details the prequisites in software and hardware if you wish to develop for the rust libraries and innovations on top of the Prusa machines. All the example code is based on being on a Linux system so you will need to translate the commands to their Windows and Mac equivalents if you are working on those platforms.

## Software

You will need the following software

- [git](https://git-scm.com/) - Version Control
- [rust](https://rust-lang.org/) - Programming Language
- [gcc](https://gcc.gnu.org/) - C/C++ compiler
- [python (3.11)](https://www.python.org/) - Used by the buddy firmware to orchestrate the build process.
- [probe-rs](https://probe.rs/) - To flash code onto the machine. Please follow their guidelines in setting up the necessary rules so you can flash the device (e.g., `udev/rules`). 
- IDE of your choice

## Printer, ST-Link Probe, USB Stick and Ethernet cable

Before we start, we need to have some kit. I used a Prusa MINI which features the Buddy Board and the majority of the components that the board supports (there are some headers for expansions boards and additional connectivity). I am aware of a software-based QEMU emulator of the MINI printer ([MINI404](https://github.com/vintagepc/MINI404)). I haven't tried it out but it may be useful for people who haven't got access to a physical buddy board.


<p align="center">
  <img src="https://blog.prusa3d.com/wp-content/uploads/2019/10/mini_on_white-640x360.jpg" style="max-height: 400px;" alt="Prusa Mini">
</p>

I also purchased an [ST-LINK/V2](https://www.st.com/en/development-tools/st-link-v2.html) in order to connect to and flash the board with code we will be writing as well as providing an RTT link that will enable us to print log messages to the terminal so we can check what is happening.

<p align="center">
  <img src="https://www.st.com/bin/ecommerce/api/image.PF251168.en.feature-description-include-personalized-no-cpn-medium.jpg" style="max-height: 400px;" alt="Appendix Close Up">
</p>

You need to install [Rust](https://www.rust-lang.org/) and [probe-rs](https://probe.rs/). Rust is the programming language and `probe-rs` provides the tools so we can communicate to the board via our ST-LINK/V2. Please go to their websites and follow their installation instructions. Make sure you follow `probe-rs`'s [setup process](https://probe.rs/docs/getting-started/probe-setup/) so your platform can communicate with the microcontroller device.

### Removing the Appendix

To enable us to flash custom software, we need to break the appendix on the board.

<p align="center">
  <img src="https://help.prusa3d.com/wp-content/uploads/2019-12-19-19_52_45-Window-800x224.jpg" style="max-height: 400px;" alt="Breaking the Appendix">
</p>

<p align="center">
  <img src="https://help.prusa3d.com/wp-content/uploads/2019-12-19-19_54_11-Window-800x520.jpg" style="max-height: 400px;" alt="Appendix Close Up">
</p>

Please read this [article](https://help.prusa3d.com/article/flashing-custom-firmware-mini-mini_14) for more information.


### Hooking up the STM32 Probe

The ST-LINK/V2 connects the PC to the buddy board. The figure below shows the necessary wiring to create the SWD link the `probe-rs` will use the write our code to the device and to provide logging through RTT so we can interrogate what is happening through the terminal.

![Pinout](https://raw.githubusercontent.com/jamesgopsill/embassy-buddy/main/img/pinout.png)


### USB stick

A USB stick will be required to store our gcode files but also the `firmware.bbf` file. Usually you would put the firmware on the USB stick and flash from their but we will be doing it directly on the chip for convenience. Regardless, the mini would still like one to be available on the stick as when we first flash directly onto the chip it will try and access the assets that are stored within it.

### Ethernet Cable

The Prusa Mini handles both ethernet and Wi-Fi (if the ESP32 expansion board is present). I use an ethernet cable connected directly between the machine and PC during development and set to link-local so the PC acts as both the client and router.
