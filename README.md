# Baofeng-UV-5RM-5RH-RE
Reverse Engineering of Baofeng UV-5RM(Global Tri-band)/UV-5RH(CN Tri-band) Radio

> [!CAUTION]  
> **Currently, This project only involves models using the BK4819 RF transceiver.**
>
> Before attempting to upgrade, please make sure that your device firmware version be at least v0.09 or above.
> Devices running firmware version below v0.09 may have different hardware design, if you  upgrade to higher version firmwares, you may encounter key function problems such as RF does not work.
>
>
> ---
> According to feedbacks, these firmware files only work on 5RB10W HARDWARE V01, which using BK4819 RF tranceiver.
>
> **Currently known BK4819 Transceiver model identification methods includes:**
> * The model name mentioned on the product description page is UV5RM, or UV5RH(M Edition)
> * The product description page confirms that it supports three-band transmission.
> * If you remove the battery, you can see a yellow round label with the word "BK" printed on it.
> 
> **If any of the following conditions are met, they may be another hardware using AT1846S transceiver:**
> * The model name is UV5RL or UV5RH (L Edition).
> * The product introduction does not mention 220-250MHz(1.25 Meters VHF) transmission support.
> * The version of firmware is v0.07 or v0.06.

> [!WARNING]  
> The firmware files under the [factory_firmware](factory_firmware/) foler are collected from different sources. 
>
> Different Hams obtaind them from their sellers and then shared them in the Ham groups or channels. 
>
> There is no guarantee that they are from the original manufacturer.
>
> These firmware files are provided for research and study purposes only.
>
> DO NOT use these firmware files for commercial purposes.
>
> DO NOT use these firmware files for any illegal purposes.


## Quick view
Front Photo
![front](./teardown/1-front.jpg)
Back Photo
![back](./teardown/2-back.jpg)
Label Photo
![labels](./teardown/5-labels.jpg)

## Teardown
Teardown Overview
![teardown](./teardown/7-teardown-2.jpg)

## Hardware
### Overview
PCB Top Photo
![pcb-top](./teardown/10-pcb-top.jpg)
PCB Bottom Photo
![pcb-bottom](./teardown/11-pcb-bottom.jpg)

### Main Components

| Component | Silkprint | Description |
| --- | --- | --- |
| MCU | SYN2A-000 | Silkprint Customized Artery AT32F421C8T7 without Artery ISP bootloader (120MHz,64KByte Flash,16KByte SRAM) |
| RF | Beken BK4819 | |
| Flash | XMC 25QH16CJIG | 16Mbit |
| LCD | (unknown yet) | |

## Reverse Engineering
### 1). MCU pinout detetion

### 2). MCU part number detection over SWD

### 3). Bootloader binary dumping

At present, Flash Access Protection was enabled by default on the device's Flash, which means that the contents of the Flash cannot be read by code running in an environment other than Flash. This also makes it impossible to run the process of dumping Flash by loading the code into RAM.

The Flash was divided to 2 areas:
> 1. The first 4KBytes were for the Bootloader
> 2. The latter 60KBytes were used to store the Firmware.

The Bootloader was used to receive the encrypted firmware upgrade packages sent from the computer, and decrypt them before writing it to the firmware area of the Flash, which is starting from the address 0x08001000.

The firmware package was encrypted, but after research, I found that the first 2KBytes of the firmware may not be encrypted, so I tried to implement a firmware smaller than 2KByte. The only function was to read the the first 4KByte of Flash which storing Bootloader and send it out through the serial port on the K connector, source code refer to [reverse_engineering/bf-uv5rm-btldr-dumper](reverse_engineering/bf-uv5rm-btldr-dumper), and the output upgrade package refer to [reverse_engineering/bf-uv5rm-btldr-dumper-fw.BF](reverse_engineering/bf-uv5rm-btldr-dumper-fw.BF).

Dumped bootloader was placed at [reverse_engineering/dumped_btldr.bin](reverse_engineering/dumped_btldr.bin)

#### [2024-02-18 UPDATE]
After some RE works on the bootlader, I found that SYSTEM BOOTLOADER region(Start from 0x1FFFE400, total 4KB erasable Flash) was used to store the configuration data, this is why the ISP bootmode of AT32F421 doesn't work.

Dumped SYSTEM BOOTLOADER was placed at [reverse_engineering/dumped_sysbtldr.bin](reverse_engineering/dumped_sysbtldr.bin) 

### 4). Bootloader reverse engineering for Firmware Upgrade Protocol and Encryption details

After the bootloader had dumped out, the details of the encryption algorithm in the bootloader were studied through Ghidra reverse engineering.

Then I prepared a group of sentences describing the algorithm details to Github Copilot, which helped me accurately generate the C language code of the encryption algorithm and the corresponding decryption code, refer to [reverse_engineering/encrypt.c](reverse_engineering/encrypt.c).

This can be used for firmware encryption and decryption, and also provides conditions for other partners who want to reverse engineer the complete firmware.

> [!NOTE]
> About the .BF wrapping file format.
>
> Here is the draft table describes the .BF file format.
> | Offset | Length | Description |
> | --- | --- | --- |
> | 0x00 | 1 | Wrapped Region Count |
> | 0x01 | 4 | Region 1 Length, Encrypted firmware bin file length  |
> | 0x05 | 4 | Region 2 Length, config length? |
> | 0x09 | 7 | No clear usage found, testing of random values seems no impact. |
> | 0x10 | Region 1 Length | Encrypted Firmware Bin Hex |
> | 0x10 + Region 1 Length | Region 2 Length | ~~Configuration- Hex? It’s not clear yet. The guess was that it is related to the configuration. It will be clear after the Bootloader and Firmware are completely reverse engineered~~ <br> Data part, this part will be upload to SYSTEM BOOTLOADER region(start from address 0x1FFFE400, maximum 4KB), this is why the ISP function of the AT32F421 doesn't work.  |

#### [2024-03-07 UPDATE]
I made a tool for wrapping/unwrapping the firmware file, refer to [reverse_engineering/uv5rm-wrap-tool](reverse_engineering/uv5rm-wrap-tool).

Thanks to OK2MOP, who also made a python script for wrapping/unwrapping the firmware file, refer to [reverse_engineering/bao_fw_tool.py](reverse_engineering/bao_fw_tool.py).

### 5). Factory firmware decryption and reverse engineering

### 6). Opensource Flashing Tool development