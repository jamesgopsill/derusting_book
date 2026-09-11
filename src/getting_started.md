
# Getting Started

So you want to jump right! Great. You can do this by downloading the pre-compiled firmware for the Prusa Mini. And don't worry about needing lots of printers. A de-centralised service can start life as a service of **one**!

### Step 1. Download the Firmware

We release pre-compiled versions of the firmware for the Prusa Mini that can be found on the releases page for this repo. All you need to do is download the `.bbf` file and load it onto the USB stick you use with your machine.

### Step 2. Removing the Appendix on the Prusa Buddy Board

The firmware is not signed so the Prusa bootloader will refuse to accept and flash it onto the device. To flash custom firmware, you need to remove the appendix on the board.

<p align="center">
  <img src="https://help.prusa3d.com/wp-content/uploads/2019-12-19-19_52_45-Window-800x224.jpg" style="max-height: 400px;" alt="Breaking the Appendix">
</p>

<p align="center">
  <img src="https://help.prusa3d.com/wp-content/uploads/2019-12-19-19_54_11-Window-800x520.jpg" style="max-height: 400px;" alt="Appendix Close Up">
</p>

Please read this [article](https://help.prusa3d.com/article/flashing-custom-firmware-mini-mini_14) for more information.

### Step 3. Flashing the firmware

Insert the USB stick into the machine and restart or turn on the machine. Click the knob multiple times during the bootloader process and it should take you to the following screen.

<p align="center">
  <img src="https://github.com/jamesgopsill/derusting_book/blob/main/src/assets/flash_device.png?raw=true" height="400" alt="Flashing Custom Firmware">
</p>

Click `FLASH` and it will attempt to confirm the signature of the firmware. Our firmware is not signed by Prusa so you will need to click `IGNORE` to continue with flashing the firmware onto the device. This will take a few moments and but you should end up at the home screen. You should see `DERUSTING` as the title of the screen and the play button, which you would typically use to print, has changed to an `OFFLINE`/`ONLINE` button.

To submit a job to the machine you need to go `http://[IP_ADDRESS_OF_MACHINE]:8080` on your network where you will be presented with:

<p align="center">
  <img src="https://github.com/jamesgopsill/derusting_book/blob/main/src/assets/website.png?raw=true" height="400" alt="Website">
</p>

You can submit your job here and the machine will accept it and save it to its USB stick. If you go back to the home screen and set the machine to `ONLINE` by toggling the button then you should see the machine spring to life as it checks the job ledger, notes it is ready to accept jobs and selects from the ledger to print.

> [!NOTE]
> The firmware is set to `dry_print` only at the moment. This restriction can be removed by compiling your own version. We will be adding a toggle for this feature in the future.

### Step 4. Celebrate!

Hooray, you've now entered the world of decentralised manufacturing. No more need for public/private/cloud servers or third-party providers. Simply connect more printers and scale your production. :smile:.

## What is happening behind the scenes.

The machines are communicating on UDP port `9090` where they broadcast their status and share the job ledger and print files when a user uploads one.

- Address Book: Each machine maintains a list of address of the other machines. Machines periodically publish their status over the wire.
- Submission Portal: Users can go to the IP address (:8080) of any of the machines where they can submit their jobs to the system. If a machine is busy it will redirect them to another machine to handle the request.
- Job sharing: Machines share the file amongst one another so it is available on all of their USB sticks for manufacture.
- Job Ledger: The machines pass around a job ledger and each get an opportunity to pick a job from the ledger to manufacture.
- OnReady Function: A machine will only take a job if a user has checked the machine and clicked the button to take it online.

If you want to listen in and see the conversation then please use our derusting udp listener.

> [!TIP]
> You may need to edit your firewall settings on your PC to listen in on the network traffic.
