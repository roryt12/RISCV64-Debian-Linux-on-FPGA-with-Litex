# RISCV64-Debian-on-FPGA-with-Litex
How to run RISCV64 Debian Linux on an FPGA with Litex

I managed to run a stable RISCV64 Debian Linux on FPGA using Litex. It has been tested with NaxRiscv and Rocket 64bit RISCV CPUs and on two FPGAs from Qmtech : qmtech_wukong (Artix xc7a100t) and qmtech_artix7_fbg484 (Artix xc7a200t plugged on vendor’s daughterboard).

Inspired from https://spinalhdl.github.io/NaxRiscv-Rtd/main/index.html (with many thanks to Charles Papon) and https://github.com/litex-hub/linux-on-litex-rocket (with many thanks to Gabriel L. Somlo).

As a development environment I used Debian 11 "bullseye" with backports. The steps I followed are (beware, YMMV):

1) I Installed Litex https://github.com/enjoy-digital/litex as usual.
```
wget https://raw.githubusercontent.com/enjoy-digital/litex/master/litex_setup.py
chmod +x litex_setup.py
./litex_setup.py --init --install --user --config=full
```
2) You will need an ***already proved toolchain*** to build the SOC, firmware and kernel. Bullseye (even from backports) has Riscv cross-compile toolchain up to version 10.2.1 and gave me a lot of headaches. Sid's is 12.2.0 and I think it is good enough. Nevertheless, I used the recommended from Litex toolchain. It takes some time to build, but worth the effort. The steps are from Litex/Rocket's Readme:

```
	git clone --recursive https://github.com/riscv/riscv-gnu-toolchain
	pushd riscv-gnu-toolchain
	./configure --prefix=$HOME/RISCV --enable-multilib
	make newlib linux -j($nproc)
	popd
```	
Put toolchain's installation bin in your $PATH and confirm that e.g. riscv64-unknown-linux-gnu-gcc runs ok.

3) In order to build the FPGA Bitstream: 

NaxRiscv on artix7_fbg484 (1 core with FPU):
```	
litex-boards/litex_boards/targets/qmtech_artix7_fbg484.py --kgates 200 --build  \
        --with-sdcard --with-daughterboard --with-ethernet  --with-video-terminal \
        --cpu-type naxriscv  --sys-clk-freq 100e6 --l2-size 0  --xlen 64 \
        --scala-args 'rvc=true,rvf=true,rvd=true,mmu=true' --uart-baudrate 3000000  \
        --csr-csv qmtech_artix7_fbg484-naxriscv.csv --csr-json qmtech_artix7_fbg484-naxriscv.json  \
        > qmtech_artix7_fbg484-naxriscv.log &  tail -f qmtech_artix7_fbg484-naxriscv.log
```
NaxRiscv on wukong (1 core with FPU):
```	
litex-boards/litex_boards/targets/qmtech_wukong.py --board-version 2 --build  \
        --with-sdcard --with-ethernet  --with-video-terminal \
        --cpu-type naxriscv  --sys-clk-freq 100e6 --l2-size 0  --xlen 64 \
        --scala-args 'rvc=true,rvf=true,rvd=true,mmu=true' --uart-baudrate 3000000  \
        --csr-csv qmtech_wukong-naxriscv.csv --csr-json qmtech_wukong-naxriscv.json  \
        > qmtech_wukong-naxriscv.log &  tail -f qmtech_wukong-naxriscv.log
```

Rocket on artix7_fbg484 (2 cores with FPU and hypervisor):
```	
litex-boards/litex_boards/targets/qmtech_artix7_fbg484.py --kgates 200 --build  \
       --with-sdcard --with-daughterboard --with-ethernet  --with-video-terminal \
       --cpu-type rocket --cpu-variant full --cpu-num-cores 2 --cpu-mem-width 2 \
       --sys-clk-freq 100e6   --l2-size 65536 --bus-bursting  --uart-baudrate 3000000 \
       --csr-csv qmtech_artix7_fbg484-rocket.csv --csr-json qmtech_artix7_fbg484-rocket.json \
       > qmtech_artix7_fbg484-rocket.log &  tail -f qmtech_artix7_fbg484-rocket.log
```	

Rocket on wukong (1 core with FPU and hypervisor):
```	
litex-boards/litex_boards/targets/qmtech_wukong.py --board-version 2 --build  \
       --with-sdcard --with-ethernet  --with-video-terminal \
       --cpu-type rocket --cpu-variant full --cpu-num-cores 2 --cpu-mem-width 2 \
       --sys-clk-freq 100e6   --l2-size 65536 --bus-bursting  --uart-baudrate 3000000 \
       --csr-csv qmtech_wukong-rocket.csv --csr-json qmtech_wukong-rocket.json \
       > qmtech_wukong-rocket.log &  tail -f qmtech_wukong-rocket.log
```	



***NOTES***
a) I changed LiteEthPHYMII to LiteEthPHYGMII, LiteEthPHY to LiteEthGMii and switched ethernet clock to 12.5MHz in litex_boards/targets/qmtech_artix7_fbg484.py and litex_boards/targets/qmtech_wukong.py. This way I managed to have a working ethernet. At least I can ssh to the board, get time synchronization, and use vncserver/vncviewer over LAN. It is at most 100MBps and after a while it needs reset  (ifdown/ifup eth0). With the original MII setup I could not make it work at all.
b) Both boards can support serial uart with speed 3MBps, hence the --uart-baudrate 3000000. Very useful if you upload images over the serial port during boot (tested with litex_term and putty)
c) The video terminal works - somehow. For some strange reason I'm getting a lot of extra white space between lines. Not sure why yet.
d) On NaxRiscv I used L2 = 0 after Charles' suggestion. On Rocket I found (after testing), that a bit larger L2 cache and bus bursting improves performance.

4) Creating the DTS / DTB :

With the recent commits it is very easy to create the DTS file, depending on the core/board, just run :
```
litex_json2dts_linux --root-device mmcblk0p3 qmtech_artix7_fbg484-naxriscv.json > qmtech_artix7_fbg484-naxriscv.dts
litex_json2dts_linux --root-device mmcblk0p3 qmtech_artix7_fbg484-rocket.json > qmtech_artix7_fbg484-rocket.dts
litex_json2dts_linux --root-device mmcblk0p3 qmtech_wukong-naxriscv.json > qmtech_wukong-naxriscv.dts
litex_json2dts_linux --root-device mmcblk0p3 qmtech_wukong-rocket.json > qmtech_wukong-rocket.dts

```
***EDIT*** your dts file and change according to your needs. See mine for comparison. ***NOTES***:
a) The reason for sbi/hvc0 in boot command line (after Charles' suggestion) is that liteuart gives me a lot of headaches, but the hvc driver works better.
b) I use “root=/dev/mmcblk0p3” because my root filesystem on the sd card is on the 3rd partition – see below
c) I modified the start/end for the initrd to ensure that fits the one I used
d) For NaxRiscv I used 
```
riscv,isa = "rv64imafdc"
```
e) I hased-out the interrupt line for liteuart, otherwise I have a lot of issues with interrupts.

f) I added a few components to the DTS files for Rocket (e.g. debug-controller etc), based on the DTS files from linux-on-litex-rocket

I also suggest to compare the values for memories/interrupts against the produced .csv / .json file to ensure that your DTS is correct.
In order to get the final DTB, run (also depending on your core/board):
```
dtc -O dtb -o qmtech_artix7_fbg484-naxriscv.dtb qmtech_artix7_fbg484-naxriscv.dts
dtc -O dtb -o qmtech_artix7_fbg484-rocket.dtb qmtech_artix7_fbg484-rocket.dts
dtc -O dtb -o qmtech_wukong-naxriscv.dtb qmtech_wukong-naxriscv.dts
dtc -O dtb -o qmtech_wukong-rocket.dtb qmtech_wukong-rocket.dts

```

5) Compile your OpenSBI binary. 

Although I can compile usable binaries for both NaxRiscv and Rocket from the latest OpenSBI repository (1.2-94), still I have issues during kernel boot, that I can not pinpoint yet. 
Instead I can produce usable binaries from an older version (0.8):
```
	git clone https://github.com/litex-hub/opensbi
	cd opensbi
	git checkout 0.8-linux-on-litex-vexriscv
	cd platform/litex
	cp -avf vexriscv naxriscv_64
              cp -avf vexriscv rocket
```		
At this point you have to edit the files in the two directories to correspond to each core. For your convenience I have included mine. 
In order to compile the binary:
NaxRiscv:
```		
cd ..
	make PLATFORM=litex/naxriscv_64 CROSS_COMPILE=riscv64-unknown-linux-gnu- -j$(nproc)
cp build/platform/litex/naxriscv_64/firmware/fw_jump.bin  ../opensbi-naxriscv.bin
```		
Rocket:
```		
cd ..
	make PLATFORM=litex/rocket CROSS_COMPILE=riscv64-unknown-linux-gnu- -j$(nproc)
cp build/platform/litex/rocket/firmware/fw_jump.bin  ../opensbi-rocket.bin
```		

6) Compile the Linux kernel  (from Rocket’s README) :

```
	git clone https://github.com/litex-hub/linux.git
	cd linux
	git checkout litex-rebase
	make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- litex_rocket_defconfig 
	
```	

You should edit .config to add a few things, i.e.
	
```
	CONFIG_RISCV_SBI=y
	CONFIG_RISCV_SBI_V01=y
	CONFIG_SERIAL_EARLYCON_RISCV_SBI=y
CONFIG_HVC_DRIVER=y
	CONFIG_HVC_RISCV_SBI=y
```
If unsure, see my .config (currently from 6.3.0-rc5).  Then run:
	
```	
	make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j$(nproc)

```

Your kernel Image will be in arch/riscv/boot/Image

7)  My target is to boot from an sdcard a full Debian environment. I created a chroot Debian (SID) environment with the instructions from https://wiki.debian.org/RISC-V, step 6, "Creating a riscv64 chroot", with debootstrap and I used it in combination with binfmt-support. 
```
	sudo apt-get install debootstrap qemu-user-static binfmt-support debian-ports-archive-keyring
	sudo debootstrap --arch=riscv64 --keyring /usr/share/keyrings/debian-ports-archive-keyring.gpg \
      --include=debian-ports-archive-keyring unstable /tmp/riscv64-chroot http://deb.debian.org/debian-ports
```

and after, run the steps in "Preparing the chroot for use in a virtual machine". The steps for u-boot are not really needed (for the moment at least). I modified a few other things in the chroot environment, e.g.  installed udev, locales / timezone and a few others. The usual things needed to be edited, eg hostname, hosts, network/interfaces, fstab etc. BEWARE with fstab, certainly you will have to adjust based on the UIDS  that your sdcard has.

In systemd's journald.conf, I used:
```
	Storage=none
```
and in logind.conf:
```
	NAutoVTs=0
	ReserveVT=0
```

and in /etc/login.defs :
```
	LOGIN_TIMEOUT           180
```

After all, this is a very low resources board and it is SLOW! I have uploaded the tarball with the filesystem for your convenience. I have included a tarball with the root filesystem, feel free to use it or use it as a guideline. It has sshd, vncserver and icewm and configures eth0 with dhcp. Root’s password is litex and vncservers’ password is litex123.

8)  I used a Debian style initrd. Initrd is a usefull to initialize devices during boot, fsck the filesystems etc. In order to make one, I used a trick: I moved my linux source tree inside /usr/src/ of my riscv Debian chroot filesystem, run chroot on the root of the filesystem, erased all scripts in the kernel tree (they have been compiled as x86 binaries) and recompiled the kernel natively with Debian's gcc, and then run make install - ie I installed the kernel inside my riscv filesystem. Debian's scripts made the initrd for me. Lazy, I know, if you know how to properly use mkinitrd feel free to use it instead. If unsure how to do any of these, use mine.

10) Created the corresponding boot.json that contains the files needed for boot.

NaxRiscv on artix7_fbg484
```
{
        "Image"				:       "0x41000000",
        "initrd.img-6.3.0-rc2"		:       "0x42000000",
        "qmtech_artix7_fbg484.dtb"	:       "0x46000000",
        "opensbi-naxriscv.bin"		:       "0x40f00000"
}
```
NaxRiscv on wukong
```
{
        "Image"				:       "0x41000000",
        "initrd.img-6.3.0-rc2"		:       "0x42000000",
        "qmtech_wukong-naxriscv.dtb"	:       "0x46000000",
        "opensbi-naxriscv.bin"		:       "0x40f00000"
}
```
Rocket on artix7_fbg484
```
{
        "Image"				:       "0x41000000",
        "initrd.img-6.3.0-rc2"		:       "0x42000000",
        "qmtech_artix7_fbg484-rocket.dtb"	:       "0x46000000",
        "opensbi-naxriscv.bin"		:       "0x40f00000"
}
```
Rocket on wukong
```
{
        "Image"				:       "0x41000000",
        "initrd.img-6.3.0-rc2"		:       "0x42000000",
        "qmtech_wukong-rocket.dts"	:       "0x46000000",
        "opensbi-naxriscv.bin"		:       "0x40f00000"
}
```
I have uploaded all 4 , copy the one you want to boot.json

11) Make an sd-card with three partitions: 1st (sda1) is VFAT that contains the above boot.json and all the other files used by it. 2nd (sda2) partition is a swap. 3rd (sda3) partition is ext4 and should contain a copy of the chroot riscv Debian filesystem mentioned above.
Ensure that fstab is correct accordingly to the card ! 

Put the card in the board, connect your vga/hdmi (optional), your ethernet and your serial port with a program that supports speed 3M (like litex_term or putty), program the bitstream with Vivado, and you are ready to boot to Debian !


