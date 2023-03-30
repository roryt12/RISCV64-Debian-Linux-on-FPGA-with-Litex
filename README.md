# RISCV64-Debian-on-FPGA-with-Litex
How to run RISCV64 Debian Linux on an FPGA with Litex

I managed to run a stable RISCV64 Debian Linux on FPGA using Litex.This has been tested with NaxRiscv and Rocket 64bit RISCV CPUs and on two FPGAs from Qmtech : qmtech_wukong (Artix xc7a100t) and qmtech_artix7_fbg484 (Artix xc7a200t) plugged qmtech's daughterboard.

Inspired from https://spinalhdl.github.io/NaxRiscv-Rtd/main/index.html (with many thanks to Charles Papon) and https://github.com/tongchen126/Boot-Debian-On-Litex-Rocket (with many thanks to Gabriel L. Somlo).

As a development environment I used Debian 11 "bullseye" with backports. The steps I followed are (beware, YMMV) :

1) I Installed Litex https://github.com/enjoy-digital/litex as usual.
```
wget https://raw.githubusercontent.com/enjoy-digital/litex/master/litex_setup.py
chmod +x litex_setup.py
./litex_setup.py --init --install --user --config=full
```
2) You will need an ***already proved toolchain*** to build the SOC,firmware and kernel. Bullseye (even from backports) has up to version 10.2.1 and gave me a lot of headaches. Sid's is 12.2.0 and I think it is good enough. Nevertheless, I choosed to use the recommended toolchain. It takes some time to build, but worth the effort. The steps are from Litex/Rocket's Readme:

```
	git clone --recursive https://github.com/riscv/riscv-gnu-toolchain
	pushd riscv-gnu-toolchain
	./configure --prefix=$HOME/RISCV --enable-multilib
	make newlib linux -j($nproc)
	popd
```	
Put toolchain's installation path in your $PATH and confirm that eg riscv64-unknown-linux-gnu-gcc runs ok.

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
a) I changed LiteEthPHYMII to LiteEthPHYGMII, LiteEthPHY to LiteEthGMii and switched ethernet clock to 12.5MHz in litex_boards/targets/qmtech_artix7_fbg484.py and litex_boards/targets/qmtech_wukong.py. This way I managed to have a working (most of the times) ethernet. At least I can ssh to the board, get time synchronization, and use vncserver/vncviewer over LAN. It is mostly 100MBps and after a while it needs reset. With the original MII setup I could not make it work at all.
b) Both boards can support serial uart with speed 3MBps, hence the --uart-baudrate 3000000. Very usufull if you upload images over the serial port during boot (tested with litex_term and putty)
c) The video terminal works - somehow. For some strange reason I'm getting a lot of extra white space between lines. Not sure why yet.
d) On NaxRiscv I used L2 = 0 after Charles' suggestion. On Rocket I found after testing, that a bit larger L2 cache and using bus bursting improves performance.

4) Creating the DTS / DTB :

With the recent commits it is very easy to create the DTS file, depending on the core/board, just run :
```
litex_json2dts_linux --root-device mmcblk0p3 qmtech_artix7_fbg484-naxriscv.json > qmtech_artix7_fbg484-naxriscv.dts
litex_json2dts_linux --root-device mmcblk0p3 qmtech_artix7_fbg484-rocket.json > qmtech_artix7_fbg484-rocket.dts
litex_json2dts_linux --root-device mmcblk0p3 qmtech_wukong-naxriscv.json > qmtech_wukong-naxriscv.dts
litex_json2dts_linux --root-device mmcblk0p3 qmtech_wukong-rocket.json > qmtech_wukong-rocket.dts

```
***EDIT*** your dts file and change according to your needs. See mine for comparison. ***NOTES***:
a) The reason for sbi/hvc0 in boot command line (after Charles' suggestion) is that liteuart gives me a lot of headaches, but the hvc driver works.
b) I modified the start/end for the initrd
c) For naxricv , in memory, I used reg = <0x41000000 0xF000000>; for simplicity and didn't used the  reserved-memory block. This is optional, I think it will work anyway with the defaults.

And now in order to get the final DTB, run (also depending your core/baord):
```
dtc -O dtb -o qmtech_artix7_fbg484-naxriscv.dtb qmtech_artix7_fbg484-naxriscv.dts
dtc -O dtb -o qmtech_artix7_fbg484-rocket.dtb qmtech_artix7_fbg484-rocket.dts
dtc -O dtb -o qmtech_wukong-naxriscv.dtb qmtech_wukong-naxriscv.dts
dtc -O dtb -o qmtech_wukong-rocket.dtb qmtech_wukong-rocket.dts

```

5) Compile your OpenSBI binary. 

For Rocket I used a suggestion from Gabriel, and used the latest repository:

```
git clone https://github.com/riscv-software-src/opensbi
cd opensbi
make -j $(nproc) CROSS_COMPILE=riscv64-unknown-linux-gnu- PLATFORM=generic FW_TEXT_START=0x80000000 FW_JUMP_ADDR=0x81000000  FW_JUMP_FDT_ADDR=0x86000000 FW_DYNAMIC=n FW_PAYLOAD=n
cp build/platform/generic/firmware/fw_jump.bin ../opensbi-rocket.bin
cd ..
```

Unfortunatelly I can not make the same with NaxRiscv, although I tried to change the parameters and/or create a new platform. Instead I used a previous version:
```
		git clone https://github.com/litex-hub/opensbi
		cd opensbi
		git checkout 0.8-linux-on-litex-vexriscv
		cd platform/litex
		cp -avf vexriscv naxriscv_64
		cd naxriscv_64
```		
Edit the files in this directory, or just copy mine from opensbi-naxriscv.tgz
```		
		cd ../..
		make PLATFORM=litex/naxriscv_64 CROSS_COMPILE=riscv64-unknown-linux-gnu- -j$(nproc)
    cp build/platform/litex/naxriscv_64/firmware/fw_jump.bin  ../opensbi-naxriscv.bin
    cd ..
```		
6) Compile the Linux kernel :

```
	git clone https://github.com/litex-hub/linux.git
	cd linux
	git checkout litex-rebase
	make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- litex_rocket_defconfig 
	
```	

You should edit .config to add a few things, ie
	
```
	CONFIG_RISCV_SBI=y
	CONFIG_RISCV_SBI_V01=y
	CONFIG_SERIAL_EARLYCON_RISCV_SBI=y
  CONFIG_HVC_DRIVER=y
	CONFIG_HVC_RISCV_SBI=y
```
If unsure, use my .config (currently from 6.3.0-rc3).  Then run:
	
```	
	make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j$(nproc)

```

Your kernel Image will be in 	arch/riscv/boot/Image

7)  My target is to boot from an sdcard a full Debian environment. I created a chroot Debian environment with the instructions from https://wiki.debian.org/RISC-V, step 6, "Creating a riscv64 chroot", with debootstrap. 
```
	sudo apt-get install debootstrap qemu-user-static binfmt-support debian-ports-archive-keyring
	sudo debootstrap --arch=riscv64 --keyring /usr/share/keyrings/debian-ports-archive-keyring.gpg \
      --include=debian-ports-archive-keyring unstable /tmp/riscv64-chroot http://deb.debian.org/debian-ports
```

and after, run the steps in "Preparing the chroot for use in a virtual machine". The steps for u-boot are not really needed (for the moment at least). I modified a few other things in the chroot environment, eg  installed udev, locales / timezone and a few others. The usual things needed to be edited, eg hostname, hosts, network/interfaces, fstab etc. BEWARE with fstab, certainly you will have to adjust based on the UIDS  that your sdcard has.

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

After all, this is a very low resources board and it is SLOW! I have uploaded the tarball with the filesystem for your convenience.


8)  I used a Debian style initrd. Initrd is a usefull to initialize devices during boot, fsck the filesystems etc. 	In order to make one, I used a trick: 
	I moved my linux source tree inside /usr/src/ of my riscv Debian chroot filesystem, run  chroot on the root of the filesystem, erased all scipts in the kernel tree (they have been compiled as x86 binaries) and recompiled the kernel natively with Debian's gcc, and	then run make install - ie I installed the kernel inside my riscv filesystem. Debian's scripts made the initrd for me. Lazy, I know, if you know how to properly use mkinitrd feel free to use it instead.
	If unsure how to do any of these, use mine.


10) Created a boot.json that contains :

```
	{
        "Image"                  :       "0x41000000",
        "qmtech_wukong.dtb"      :       "0x46000000",
        "initrd.img"   		 :       "0x42000000",
        "opensbi.bin"            :       "0x40f00000"
	}

```

11) Prepared a switable sdcard with three partitions: 1st is VFAT that contains the above boot.json and all the other files used by it. 2nd partition is a swap. 3rd partition is ext4 and should contain a copy of the chroot riscv Debian filesystem.
Ensure that fstab is correct accordingly to the card ! 


Put the card in the board, program the bitstream with Vivado, and you are ready to boot to Debian !
