# 一般流程

## 环境变量设置

```bash
source /opt/Xilinx/PetaLinux/2022.2/tool/settings.sh
```

## `xsa` 位置

```bash
mkdir hardware
# mv xsa to hardware folder
```

## 产生项目文件(工作目录)

```bash
petalinux-create -t project -n petalinux --template zynqMP
```

## 导入硬件信息

petalinux是工作目录

```bash
cd petalinux/
petalinux-config --get-hw-description ../hardware/
```

如果不想修改`project-spec/configs/config`, 也可以跳过`menuconfig`

```bash
petalinux-config --silentconfig --get-hw-description=../hardware/
```

更新 xsa 文件后，需要这样

```bash
petalinux-build -x mrproper
petalinux-build
```
避免旧缓存导致 DTS 不更新的问题


## 配置本地 sstate 和 downloads 目录

```bash
petalinux-config
```

```bash
Yocto Settings  --->
    Local sstate feeds settings  --->
        (file:///opt/Xilinx/PetaLinux/2022.2/sstate-cache) Local sstate feeds URL
    Add pre-mirror url  --->
        (file:///opt/Xilinx/PetaLinux/2022.2/downloads) Add pre-mirror url
```

原来的 pre-mirror url 是 `http://petalinux.xilinx.com/sswreleases/rel-v${PETALINUX_MAJOR_VER}/downloads`

体现在

```bash
build/conf/plnxtool.conf
```




## proxy等其他环境变量

```bash
export http_proxy="127.0.0.1:8118"
export https_proxy="127.0.0.1:8118"
export PETALINUX_SSTATE_LOC=/opt/Xilinx/PetaLinux/2022.2/sstate-cache
export PETALINUX_DOWNLOADS_LOC=/opt/Xilinx/PetaLinux/2022.2/downloads
//export YOCTO_NO_NETWORK=1
```


## 进一步配置
```bash
petalinux-config -c u-boot
petalinux-config -c kernel
petalinux-config -c rootfs
```

## 开始构建
```bash
petalinux-build
```
清理
```bash
petalinux-build -x clean
petalinux-build -x cleansstate
petalinux-build -x mrproper             # 最彻底
```

## 构建完毕后产生目标文件
```bash
petalinux-package --boot --u-boot --fpga --force
```

默认就是sd卡启动ramfs

`petalinux/images/linux`目录下的这三个文件放到vfat格式的sd卡就可以。记得启动拨码给对。

```bash
boot.scr
BOOT.BIN
image.ub
```

## 产生sdk
```bash
petalinux-build --sdk
```
输出在
```bash
build/tmp/deploy/sdk
```










# 问题处理1: PL设备节点名修正

`petalinux/components/plnx_workspace/device-tree/device-tree/pl.dtsi`自动产生endpoint名称有错误，

```bash
/home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta/petalinux/project-spec/configs/../../components/plnx_workspace/device-tree/device-tree/pl.dtsi:104.66-106.8: ERROR (phandle_references): /amba_pl@0/mipi_csi2_rx_subsystem@80050000/ports/port@1/endpoint: Reference to non-existent node or label "mipi_csi2_rx_axis_passthrough_mon_0mipi_csi2_rx_mipi_csi2_rx_subsyst_0"

/home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta/petalinux/project-spec/configs/../../components/plnx_workspace/device-tree/device-tree/pl.dtsi:206.48-208.8: ERROR (phandle_references): /amba_pl@0/v_proc_ss@80080000/ports/port@1/endpoint: Reference to non-existent node or label "mipi_csi2_rx_axis_switch_0mipi_csi2_rx_v_proc_ss_0"

/home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta/petalinux/project-spec/configs/../../components/plnx_workspace/device-tree/device-tree/pl.dtsi:243.44-245.8: ERROR (phandle_references): /amba_pl@0/v_tpg@800c0000/ports/port@1/endpoint: Reference to non-existent node or label "mipi_csi2_rx_axis_switch_0mipi_csi2_rx_v_tpg_0"
```

修改为正确的，继续编译，怎么修改呢？


## test0: 启用`dt overlay`
```bash
petalinux-build -c device-tree -x cleansstate
petalinux-build -c device-tree -x do_compile

petalinux-build -x cleansstate  == bitbake petalinux-image-minimal -c cleansstate
```

能编译通过，但是有啥作用呢？ 还是关闭掉，重置默认关闭

修改petalinux-config之后最有效的重置方法
```
petalinux-build -x clean        == bitbake petalinux-image-minimal -c clean
```

按理说只是device-tree这样也应该是彻底重置

```
petalinux-build -c device-tree -x clean
```



## test1: 先去掉`endpoint`
petalinux/project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi
```dtd
/*Add pl custom nodes for pl.dtsi which is generated from base xsa file.
Changes in this file reflects only when enabled the FPGA manager/Device tree overlay.*/
/ {
    user_test_property = <12345>;
};

&mipi_csi2_rx_mipi_csi2_rx_subsyst_0 {
    ports {
        // 覆盖 port@1（去掉 remote-endpoint 和 endpoint）
        port@1 {
            reg = <1>;
            xlnx,cfa-pattern = "rggb";
            xlnx,video-format = <12>;
            xlnx,video-width = <8>;
        };

        // 保留 port@0 并保持 data-lanes
        port@0 {
            reg = <0>;
            xlnx,cfa-pattern = "rggb";
            xlnx,video-format = <12>;
            xlnx,video-width = <8>;

            mipi_csi_in: endpoint {
                data-lanes = <1 2>;
            };
        };
    };
};
```

有命令
```shell
petalinux-build -c device-tree -x cleansstate
petalinux-build -c device-tree == petalinux-build -c device-tree -x do_compile
petalinux-build

grep -R "mipi" build/tmp/work/*/device-tree/*/system-top.dtb
find . -type f -name "system-top.dtb"
```

## test2: 补全`endpoint`
在git内体现







# 问题处理2: 没有检测到i2c设备
```bash
root@petalinux:~# i2cdetect -y 0  
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --                         
root@petalinux:~# i2cdetect -y 1  
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --                         
root@petalinux:~# i2cdetect -y 2
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --        

root@petalinux:~# i2cdetect -l  
i2c-0   i2c             i2c-gpio-0                              I2C adapter
i2c-1   i2c             i2c-gpio-1                              I2C adapter
i2c-2   i2c             i2c-gpio-2                              I2C adapter

```

下面修改无效
```bash
&xgpio_i2c_0_axi_gpio_0 {
    xlnx,all-inputs = <0x0>;
    xlnx,all-outputs = <0xFFFFFFFF>;    /* 全部配置为输出 */
    xlnx,dout-default = <0xFFFFFFFF>;   /* 上电后的默认电平为高 */
    xlnx,tri-default = <0x0>;
};
```

```bash
&xgpio_i2c_0_axi_gpio_0 {
    #gpio-cells = <2>;
    gpio-controller;

    compatible = "xlnx,axi-gpio-2.0", "xlnx,xps-gpio-1.00.a";
    reg = <0x0 0x80110000 0x0 0x10000>;

    xlnx,gpio-width = <6>;
    xlnx,is-dual = <0>;

    /* 必须：全部 input，允许 i2c-gpio 控制 */
    xlnx,tri-default = <0xFFFFFFFF>;
};
```


有关命令
```bash
i2cdetect -y 1
i2cget -f -y 1 <addr> <reg>
i2cset -f -y 1 <addr> <reg> <val>
i2ctransfer -f -y 1 w1@<addr> <reg> r1
i2ctransfer -f -y 1 w2@0x6c 0x00 0x00 r1

```


板子上暂时也查看不了波形，放弃使用gpio-i2c




# 更换为`axi-i2c`

## 问题1: 没有出现i2c节点
```bash
root@petalinux:~# i2cdetect -l
root@petalinux:~# i2cdetect -l
root@petalinux:~# ls /dev/i* -l
crw-------    1 root     root      246,   0 Mar 24 10:32 /dev/iio:device0
lrwxrwxrwx    1 root     root            12 Mar 24 10:32 /dev/initctl -> /run/initc
```

```bash
root@petalinux:~# dmesg | grep i2c
[    4.931796] i2c_dev: i2c /dev entries driver
[    5.580322] xiic-i2c 80010000.i2c: IRQ index 0 not found
[    5.591610] xiic-i2c 80020000.i2c: IRQ index 0 not found
[    5.604501] xiic-i2c 80030000.i2c: IRQ index 0 not found
```

看起来是缺少中断号，那么vivado把中断连上

测试ok，能检测到




# 从开发板到目标板

目标板没有tf卡插口，所以，设想先从qspi_flash启动ramfs。64MB的大小，需要55MB，能装下。

这个qspi启动的版本，可以用来mount usb设备然后刷emmc，emmc也分区，刷好后从emmc启动即可。比没有sd卡的其实费事一点。


## 一些文件的作用
```bash
image.ub: 包括内核、ramdisk、设备树dtb
u-boot.elf: 通用boot文件，负责将image.ub从flash、sd卡等load到DDR中
zynqmp_fsbl.elf: 加载bitstream，初始化DDR，初始化时钟PLL等，加载u-boot
```

## 打包产生BOOT.bin
```bash
petalinux-package --boot --fsbl --fpga --u-boot --kernel --force
```

### 报错
```bash
$ petalinux-package --boot --fsbl --fpga --u-boot --kernel --force
[INFO] Sourcing buildtools
INFO: Getting system flash information...
INFO: File in BOOT BIN: "/home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta1/petalinux/images/linux/zynqmp_fsbl.elf"
INFO: File in BOOT BIN: "/home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta1/petalinux/images/linux/pmufw.elf"
INFO: File in BOOT BIN: "/home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta1/petalinux/project-spec/hw-description/system_wrapper.bit"
INFO: File in BOOT BIN: "/home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta1/petalinux/images/linux/bl31.elf"
INFO: File in BOOT BIN: "/home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta1/petalinux/images/linux/system.dtb"
INFO: File in BOOT BIN: "/home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta1/petalinux/images/linux/u-boot.elf"
INFO: File in BOOT BIN: "/home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta1/petalinux/images/linux/image.ub"
INFO: File in BOOT BIN: "/home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta1/petalinux/images/linux/boot.scr"
INFO: Generating zynqmp binary package BOOT.BIN...


****** Xilinx Bootgen v2022.2
  **** Build date : Sep 26 2022-06:24:42
    ** Copyright 1986-2022 Xilinx, Inc. All Rights Reserved.

[ERROR]  : Section image.ub.0 offset of 0x100000 overlaps with prior section end address of 0xCD5880
ERROR: Fail to create BOOT image
```


修改flash分区配置
```bash
petalinux-config
```
增加到`0xD00000`,然后
```bash
petalinux-build
```

生成烧写文件
```bash
petalinux-package --boot --fsbl --fpga --u-boot --kernel --force
du -b images/linux/BOOT.BIN | awk '{print substr($1,$2)}' | xargs -I {} printf "%x\n" {} 
3e80adc
```

烧写, 新开一个terminal
```bash
source /opt/Xilinx/Vitis/2022.2/settings64.sh
program_flash -f images/linux/BOOT.BIN -offset 0 -flash_type qspi-x8-dual_parallel -fsbl images/linux/zynqmp_fsbl.elf -url TCP:127.0.0.1:3121 
```

板卡输出, 超出了?
```bash
SF: Detected w25q256fw with page size 512 Bytes, erase size 128 KiB, total 64 MiB
Size exceeds partition or device limit
sf - SPI flash sub-system

Usage:
sf probe [[bus:]cs] [hz] [mode] - init flash device on given SPI bus
                                  and chip select
sf read addr offset|partition len       - read `len' bytes starting at
                                          `offset' or from start of mtd
                                          `partition'to memory at `addr'
sf write addr offset|partition len      - write `len' bytes from memory
                                          at `addr' to flash at `offset'
                                          or to start of mtd `partition'
sf erase offset|partition [+]len        - erase `len' bytes from `offset'
                                          or from start of mtd `partition'
                                         `+len' round up `len' to block size
sf update addr offset|partition len     - erase and write `len' bytes from memory
                                          at `addr' to flash at `offset'
                                          or to start of mtd `partition'
sf protect lock/unlock sector len       - protect/unprotect 'len' bytes starting
                                          at address 'sector'

```

`64×1024×1024=0x4000000`, 没有超出

自动产生的bootgen.bif文件
```
the_ROM_image:
{
	[bootloader, destination_cpu=a53-0] /home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta1/petalinux/images/linux/zynqmp_fsbl.elf
	[pmufw_image] /home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta1/petalinux/images/linux/pmufw.elf
	[destination_device=pl] /home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta1/petalinux/project-spec/hw-description/system_wrapper.bit
	[destination_cpu=a53-0, exception_level=el-3, trustzone] /home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta1/petalinux/images/linux/bl31.elf
	[destination_cpu=a53-0, load=0x00100000] /home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta1/petalinux/images/linux/system.dtb
	[destination_cpu=a53-0, exception_level=el-2] /home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta1/petalinux/images/linux/u-boot.elf
	[destination_cpu=a53-0, offset=0xd00000, partition_owner=uboot] /home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta1/petalinux/images/linux/image.ub
	[destination_cpu=a53-0, offset=0x3E80000, partition_owner=uboot] /home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_vdma_peta1/petalinux/images/linux/boot.scr
}
```

其实qspi里面的工程不需要pl部分的, 新建一个最小系统来放到qspi_flash吧


# 最小系统

`zirui/04_hdmi_tx/mini`工程产生和验证了`xsa`文件

先按一般流程进行编译

然后生成烧写文件

## 打包命令

```bash
petalinux-package --boot --fsbl --u-boot --kernel --force
```

需要修改 `CONFIG_SUBSYSTEM_FLASH_PSU_QSPI_0_BANKLESS_PART0_SIZE=0x1B0000`
```bash
$ petalinux-package --boot --fsbl --u-boot --kernel --force
[INFO] Sourcing buildtools
INFO: Getting system flash information...
INFO: File in BOOT BIN: "/home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/zynqmp_fsbl.elf"
INFO: File in BOOT BIN: "/home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/pmufw.elf"
INFO: File in BOOT BIN: "/home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/bl31.elf"
INFO: File in BOOT BIN: "/home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/system.dtb"
INFO: File in BOOT BIN: "/home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/u-boot.elf"
INFO: File in BOOT BIN: "/home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/image.ub"
INFO: File in BOOT BIN: "/home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/boot.scr"
INFO: Generating zynqmp binary package BOOT.BIN...


****** Xilinx Bootgen v2022.2
  **** Build date : Sep 26 2022-06:24:42
    ** Copyright 1986-2022 Xilinx, Inc. All Rights Reserved.

[ERROR]  : Section image.ub.0 offset of 0x100000 overlaps with prior section end address of 0x1A4880
ERROR: Fail to create BOOT image
```
其实还是超大, 怎么还是和前面那个一样大呢?
```bash
du -b images/linux/BOOT.BIN | awk '{print substr($1,$2)}' | xargs -I {} printf "0x%x\n" {} 
0x3e80adc
```

计算一下
```bash
zynqmp_fsbl.elf     462.4KB
pmufw.elf           496.6KB
bl31.elf            152.5KB
system.dtb           37.7KB
u-boot.elf            9.3MB
image.ub             35.4MB
boot.scr              2.8KB
```
总大小肯定没有超过

自动产生的bootgen.bif文件
```bash
the_ROM_image:
{
	[bootloader, destination_cpu=a53-0] /home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/zynqmp_fsbl.elf
	[pmufw_image] /home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/pmufw.elf
	[destination_cpu=a53-0, exception_level=el-3, trustzone] /home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/bl31.elf
	[destination_cpu=a53-0, load=0x00100000] /home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/system.dtb
	[destination_cpu=a53-0, exception_level=el-2] /home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/u-boot.elf
	[destination_cpu=a53-0, offset=0x1b0000, partition_owner=uboot] /home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/image.ub
	[destination_cpu=a53-0, offset=0x3E80000, partition_owner=uboot] /home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/boot.scr
}
```
最后那个boot.scr给的地址太大了

## 显示某文件的十六进制大小

```bash
du -b images/linux/image.ub | awk '{print substr($1,$2)}' | xargs -I {} printf "0x%x\n" {} 
0x21ba510
```

计算boot.scr偏移
```bash
1b0000+21ba510=236A510
```
取0x2400000

修改bootgen.bif文件
```bash
the_ROM_image:
{
	[bootloader, destination_cpu=a53-0] /home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/zynqmp_fsbl.elf
	[pmufw_image] /home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/pmufw.elf
	[destination_cpu=a53-0, exception_level=el-3, trustzone] /home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/bl31.elf
	[destination_cpu=a53-0, load=0x00100000] /home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/system.dtb
	[destination_cpu=a53-0, exception_level=el-2] /home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/u-boot.elf
	[destination_cpu=a53-0, offset=0x1b0000, partition_owner=uboot] /home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/image.ub
	[destination_cpu=a53-0, offset=0x2400000, partition_owner=uboot] /home/andy/workdir/zirui/04_hdmi_tx/mini_peta/petalinux/images/linux/boot.scr
}

```
## 通过vitis命令产生 BOOT.BIN

新开一个terminal

```bash
source /opt/Xilinx/Vitis/2022.2/settings64.sh
bootgen -image images/linux/bootgen.bif -arch zynqmp -o images/linux/BOOT.BIN
du -b images/linux/BOOT.BIN | awk '{print substr($1,$2)}' | xargs -I {} printf "0x%x\n" {} 
0x2400adc
```
这次大小总算合适了

## 通过vitis命令烧写

新开一个terminal

```bash
source /opt/Xilinx/Vitis/2022.2/settings64.sh
program_flash -f images/linux/BOOT.BIN -offset 0 -flash_type qspi-x8-dual_parallel -fsbl images/linux/zynqmp_fsbl.elf -url TCP:127.0.0.1:3121 
```

这次显示的
```bash
Xilinx Zynq MP First Stage Boot Loader 
Release 2022.2   Oct  7 2022  -  04:56:16
NOTICE:  BL31: v2.6(release):xlnx_rebase_v2.6_2022.1_update3-18-g0897efd45
NOTICE:  BL31: Built : 03:55:03, Sep  9 2022

```

正常应该
```bash
Xilinx Zynq MP First Stage Boot Loader 
Release 2022.2   Oct  7 2022  -  04:56:16
NOTICE:  BL31: v2.6(release):xlnx_rebase_v2.6_2022.1_update3-18-g0897efd45
NOTICE:  BL31: Built : 03:55:03, Sep  9 2022


U-Boot 2022.01 (Sep 20 2022 - 06:35:33 +0000)

CPU:   ZynqMP
Silicon: v3
Board: Xilinx ZynqMP
DRAM:  4 GiB
PMUFW:  v1.1
PMUFW no permission to change config object
EL Level:       EL2
Chip ID:        zu7ev
NAND:  0 MiB
MMC:   mmc@ff160000: 0, mmc@ff170000: 1
Loading Environment from SPIFlash...
```
也就是 u-boot 没有启动


对比bootgen.bif文件发现0x3E80000是固定的, `grep -R 3E80000` 

```bash
build/tmp/work/x86_64-linux/qemu-xilinx-system-native/v6.1.0-xilinx-v2022.2+gitAUTOINC+74d70f8008-r0/git/roms/u-boot/board/xilinx/Kconfig:	default 0x3E80000 if ARCH_ZYNQMP
build/tmp/work/zynqmp_generic-xilinx-linux/u-boot-xlnx/v2021.01-xilinx-v2022.2+gitAUTOINC+b31476685d-r0/build/source/board/xilinx/Kconfig:	default 0x3E80000 if ARCH_ZYNQMP
build/tmp/work/zynqmp_generic-xilinx-linux/u-boot-xlnx/v2021.01-xilinx-v2022.2+gitAUTOINC+b31476685d-r0/build/include/generated/autoconf.h:#define CONFIG_BOOT_SCRIPT_OFFSET 0x3E80000
build/tmp/work/zynqmp_generic-xilinx-linux/u-boot-xlnx/v2021.01-xilinx-v2022.2+gitAUTOINC+b31476685d-r0/build/include/config/auto.conf:CONFIG_BOOT_SCRIPT_OFFSET=0x3E80000
build/tmp/work/zynqmp_generic-xilinx-linux/u-boot-xlnx/v2021.01-xilinx-v2022.2+gitAUTOINC+b31476685d-r0/build/.config.old:CONFIG_BOOT_SCRIPT_OFFSET=0x3E80000
build/tmp/work/zynqmp_generic-xilinx-linux/u-boot-xlnx/v2021.01-xilinx-v2022.2+gitAUTOINC+b31476685d-r0/build/spl/u-boot.cfg:#define CONFIG_BOOT_SCRIPT_OFFSET 0x3E80000
build/tmp/work/zynqmp_generic-xilinx-linux/u-boot-xlnx/v2021.01-xilinx-v2022.2+gitAUTOINC+b31476685d-r0/build/.config:CONFIG_BOOT_SCRIPT_OFFSET=0x3E80000
build/tmp/work/zynqmp_generic-xilinx-linux/u-boot-xlnx/v2021.01-xilinx-v2022.2+gitAUTOINC+b31476685d-r0/build/u-boot.cfg:#define CONFIG_BOOT_SCRIPT_OFFSET 0x3E80000
build/tmp/work/zynqmp_generic-xilinx-linux/u-boot-xlnx/v2021.01-xilinx-v2022.2+gitAUTOINC+b31476685d-r0/git/board/xilinx/Kconfig:	default 0x3E80000 if ARCH_ZYNQMP

```
可见, u-boot 的 CONFIG_BOOT_SCRIPT_OFFSET 是关键

```bash
petalinux-config -c u-boot
```
修改`Boot script offset`为`0x2400000`


```bash
petalinux-build -c u-boot -x clean
petalinux-build -c u-boot -x do_compile
petalinux-package --boot --fsbl --u-boot --kernel --force
```
bootgen.bif文件没有变


```bash
petalinux-build -x mrproper
petalinux-build

```

bootgen.bif文件还是没有变

实际上`0x3E80000`也没有超出`64MB`的范围, 再分析启动过程

```bash
Hit any key to stop autoboot:  0 
SF: Detected w25q256fw with page size 512 Bytes, erase size 128 KiB, total 64 MiB
device 0 offset 0x3e80000, size 0x80000
SF: 524288 bytes @ 0x3e80000 Read: OK
QSPI: Trying to boot script at 20000000
## Executing script at 20000000
Trying to load boot images from qspi0
SF: Detected w25q256fw with page size 512 Bytes, erase size 128 KiB, total 64 MiB
Size exceeds partition or device limit
sf - SPI flash sub-system

Usage:
sf probe [[bus:]cs] [hz] [mode] - init flash device on given SPI bus
                                  and chip select
sf read addr offset|partition len       - read `len' bytes starting at
                                          `offset' or from start of mtd
                                          `partition'to memory at `addr'
sf write addr offset|partition len      - write `len' bytes from memory
                                          at `addr' to flash at `offset'
                                          or to start of mtd `partition'
sf erase offset|partition [+]len        - erase `len' bytes from `offset'
                                          or from start of mtd `partition'
                                         `+len' round up `len' to block size
sf update addr offset|partition len     - erase and write `len' bytes from memory
                                          at `addr' to flash at `offset'
                                          or to start of mtd `partition'
sf protect lock/unlock sector len       - protect/unprotect 'len' bytes starting
                                          at address 'sector'

sf test offset len              - run a very basic destructive test
Wrong Image Format for bootm command
ERROR: can't get kernel image!
Booting using Fit image failed
device 0 offset 0xf00000, size 0x1d00000
SF: 30408704 bytes @ 0xf00000 Read: OK
Offset exceeds device limit
```

再看 `boot.scr`

```bash
	if test "${boot_target}" = "xspi0" || test "${boot_target}" = "qspi" || test "${boot_target}" = "qspi0"; then
		sf probe 0 0 0;
		sf read 0x10000000 0xF40000 0x6400000
		bootm 0x10000000;
		echo "Booting using Fit image failed"

		sf read 0x00200000 0xF00000 0x1D00000
		sf read 0x04000000 0x4000000 0x4000000
		booti 0x00200000 0x04000000 0x00100000;
		echo "Booting using Separate images failed"
	fi
```

也就是说, 修改config的u-boot菜单里的偏移量就可以呗

## 可以通过u-boot命令进行验证`boot.scr`

```bash
		sf probe 0 0 0;
		sf read 0x10000000 0x1B0000 0x2200000
		bootm 0x10000000;
```

验证通过

## 总结

打包命令 会产生 `petalinux/images/linux/bootgen.bif` 文件, 核对其中偏移量

一般需要修改这几个
```bash
CONFIG_SUBSYSTEM_FLASH_PSU_QSPI_0_BANKLESS_PART0_SIZE=0x1B0000
CONFIG_SUBSYSTEM_UBOOT_QSPI_FIT_IMAGE_OFFSET=0x1B0000
CONFIG_SUBSYSTEM_UBOOT_QSPI_FIT_IMAGE_SIZE=0x2200000
```
`u-boot`的`Boot script offset`在2022.2这个版本覆盖定义之后似乎没有起作用, 如果实在要重定义, 手动用`vitis`命令配合修改的`bootgen.bif`产生打包文件

`fitimage_name=image.ub`, 而且要核对`petalinux/images/linux/boot.scr`对应启动方式的偏移量和大小. 可以通过`u-boot`有关命令进行验证

通过`petalinux-config`流程 或者 手动修改再通过等效命令 都可以产生最终的烧写文件.

目的是为了对`emmc`进行刷写(目标板子没有留`sd`卡接口).



## Zynq UltraScale+ MPSoC 启动模式  — 4‑bit BOOT_MODE[3:0] 拨码表

| 4‑bit BOOT_MODE | 启动源名称   | 说明                                                    |
| --------------- | ------------ | ------------------------------------------------------- |
| `0000`          | JTAG         | Host JTAG 下载或调试启动                                |
| `0001`          | QSPI24       | Quad‑SPI Flash, 24‑bit 地址                             |
| `0010`          | QSPI32       | Quad‑SPI Flash, 32‑bit 地址                             |
| `0011`          | SD0 (2.0)    | SD 卡接口 0 (SD 2.0)                                    |
| `0100`          | NAND         | NAND Flash 启动                                         |
| `0101`          | SD1 (2.0)    | SD 卡接口 1 (SD 2.0)                                    |
| `0110`          | eMMC (1.8V)  | eMMC 启动 (1.8V signaling)                              |
| `0111`          | USB (2.0)    | USB DFU 设备引导                                        |
| `1000`          | PJTAG MIO#0  | Parallel/JTAG through MIO #0 (特定板载选择，用于 debug) |
| `1001`          | PJTAG MIO#1  | Parallel/JTAG through MIO #1                            |
| `1010`          | (保留)       | 未定义/保留                                             |
| `1011`          | (保留)       | 未定义/保留                                             |
| `1100`          | SD1 (3.0 LS) | SD 3.0 with level shifter                               |
| `1101`          | (保留)       | 未定义/扩展                                             |
| `1110`          | (保留)       | 未定义/扩展                                             |
| `1111`          | (保留)       | 未定义/扩展                                             |

> 1. **低 3 位（BOOT_MODE[2:0]）是 Boot ROM 真正读取的核心引脚数值**，高位 BOOT_MODE[3] 是用于板级扩展/记录不同接口选择。  
> 2. QSPI24/32 → 表示不同宽度的 QSPI Flash 地址宽度模式（24‑bit / 32‑bit），对于生成 Boot 镜像和 Bootgen 参数不同。 :contentReference[oaicite:1]{index=1}  
> 3. SD0/SD1 对应板子上不同 SD 接口的 MIO 引脚组合。 `SD1 (3.0 LS)` 是带电平转换支持 SD 3.0 卡的模式。 :contentReference[oaicite:2]{index=2}  
> 4. USB 引导是采用 DFU（device firmware upgrade）标准方式。 :contentReference[oaicite:3]{index=3}

## 示例拨码说明（常见板子/跳线）

| 设备/板子          | 4‑bit 设置 | 启动模式    | 备注             |
| ------------------ | ---------- | ----------- | ---------------- |
| 默认 SD 卡启动     | `0011`     | SD0 Boot    | SD 卡接口 0      |
| 用 QSPI Flash 启动 | `0010`     | QSPI32 Boot | 32‑bit 地址 QSPI |
| JTAG 调试下载      | `0000`     | JTAG Boot   | BootROM 等待下载 |
| eMMC 启动          | `0110`     | eMMC Boot   | 通常用于产品固化 |

> 实际硬件上，根据板载开关/SW6/SW8 标识，将 BOOT_MODE[3:0] 连接到 PS MODE 引脚（MIO25/MIO … 等）即可。不同板子标注可能略微不同，按上表理解最核心的低 3 位即可。 :contentReference[oaicite:4]{index=4}

# `IDT_8T49N241`地址问题

`datasheet`中有这样的描述

```
These values are specific to the device configuration and can be customized when ordering. Generic dash codes -900 through -902,
-998 and -999 are available and programmed with the default I2C address of 1111100b. Please refer to the FemtoClock NG Universal
Frequency Translator Ordering Product Information guide for more details.
```

那么默认地址就是 7-bit 的 0x7c

规范上属于`reserved for future use`，但 `Renesas`故意选这个地址来避免总线冲突。合法，但不建议普通设备使用。



## `xgpio`引脚进行复位

### 先确定引脚号

```bash
root@petalinux:~# ls -l /sys/class/gpio
total 0
--w-------    1 root     root          4096 Mar 26 05:03 export
lrwxrwxrwx    1 root     root             0 Mar 26 05:03 gpiochip302 -> ../../devices/platform/axi/ff0a0000.gpio/gpio/gpiochip302
lrwxrwxrwx    1 root     root             0 Mar 26 05:03 gpiochip476 -> ../../devices/platform/amba_pl@0/80100000.gpio/gpio/gpiochip476
lrwxrwxrwx    1 root     root             0 Mar 26 05:03 gpiochip508 -> ../../devices/platform/firmware:zynqmp-firmware/firmware:zynqmp-firmware:gpio/gpio/gpiochip508
--w-------    1 root     root          4096 Mar 26 05:03 unexport



ls /proc/device-tree/amba_pl\@0/gpio\@80100000/





hexdump -C /proc/device-tree/amba_pl\@0/gpio\@80100000/xlnx\,tri-default
hexdump -C /proc/device-tree/amba_pl\@0/gpio\@80100000/xlnx\,all-outputs




ls /proc/device-tree/amba_pl@0/ | grep gpio




grep -R "80100000" /proc/device-tree/__symbols__                                                                               
/proc/device-tree/__symbols__/rest_gpio:/amba_pl@0/gpio@80100000




root@petalinux:~# cat /sys/class/gpio/gpiochip476/label 
80100000.gpio






for c in gpiochip470 gpiochip476 gpiochip508 gpiochip296; do
  echo "=== $c ==="
  cat /sys/class/gpio/$c/base
  cat /sys/class/gpio/$c/ngpio
  cat /sys/class/gpio/$c/label 2>/dev/null || true
  readlink -f /sys/class/gpio/$c/device
  echo
done


=== gpiochip476 ===
476
32
80100000.gpio
/sys/devices/platform/amba_pl@0/80100000.gpio

```



### 导出节点

导出节点（如果还没导出）到用户空间

```bash
echo 476 > /sys/class/gpio/export
```

这样就生成了`/sys/class/gpio/gpio476`节点

```bash
root@petalinux:~# ls /sys/class/gpio
export       gpio476      gpiochip302  gpiochip476  gpiochip508  unexport
```

### 设置输入输出方向

```bash
echo out > /sys/class/gpio/gpio476/direction

cat /sys/class/gpio/gpio476/direction
```

### 设置输出值

```bash
echo 1 > /sys/class/gpio/gpio476/value   # 拉高
echo 0 > /sys/class/gpio/gpio476/value   # 拉低

cat /sys/class/gpio/gpio476/value
```

## 其他的几个复位 pin

```bash
echo 477 > /sys/class/gpio/export
echo 478 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio477/direction
echo out > /sys/class/gpio/gpio478/direction
echo 1 > /sys/class/gpio/gpio477/value
echo 1 > /sys/class/gpio/gpio478/value
cat /sys/class/gpio/gpio477/direction
cat /sys/class/gpio/gpio477/value
cat /sys/class/gpio/gpio478/direction
cat /sys/class/gpio/gpio478/value
```

## 扫描总线

复位设备之后扫描总线

地址是0x7c, 这样i2cdetect是访问不了吗? `-a` 就可以访问了

```bash
root@petalinux:/sys/class/gpio# i2cdetect -y -a 0
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- 3c -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
root@petalinux:/sys/class/gpio# i2cdetect -y -a 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- -- -- -- -- -- 7c -- -- -- 
root@petalinux:/sys/class/gpio# i2cdetect -y -a 2
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- 5c -- 5e -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
```

## 访问设备

## 用`i2ctransfer`而不是`i2cget/i2cset`

读 16-bit 地址 + 8-bit 数据

```bash
root@petalinux:/sys/class/gpio# i2ctransfer -y -a 1 w2@0x7c 0x00 0x03 r1
0x60
```

读 **16-bit 地址 + 16-bit 数据**

```bash
root@petalinux:/sys/class/gpio# i2ctransfer -y -a 1 w2@0x7c 0x00 0x02 r2
0x00 0x60
```

读 寄存器地址 `0x0020` 起的长度 4字节

```bash
i2ctransfer -y -a 1 w2@0x7c 0x00 0x20 r4
```

16-bit寄存器地址写单字节

```bash
i2ctransfer -y -a 1 w3@0x7c 0x00 0x20 0x5A
```

16-bit寄存器地址写双字节

```bash
i2ctransfer -y -a 1 w4@0x7c 0x00 0x20 0x12 0x34
```

i2cget 只能发送 8-bit寄存器地址, 无法发送 16-bit寄存器地址。 所以

```bash
root@petalinux:/sys/class/gpio# i2cget -f -y -a 1 0x7c 0x00 0x03
Error: Invalid mode!
Usage: i2cget [-f] [-y] [-a] I2CBUS CHIP-ADDRESS [DATA-ADDRESS [MODE [LENGTH]]]
  I2CBUS is an integer or an I2C bus name
  ADDRESS is an integer (0x08 - 0x77, or 0x00 - 0x7f if -a is given)
  MODE is one of:
    b (read byte data, default)
    w (read word data)
    c (write byte/read byte)
    s (read SMBus block data)
    i (read I2C block data)
    Append p for SMBus PEC
  LENGTH is the I2C block data length (between 1 and 32, default 32)
```

目标板上也进行类似操作即可

```bash
root@petalinux:~# echo 476 > /sys/class/gpio/export
root@petalinux:~# echo out > /sys/class/gpio/gpio476/direction
root@petalinux:~# echo 1 > /sys/class/gpio/gpio476/value
root@petalinux:~# i2cdetect -y -a 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- -- -- -- -- -- 7c -- -- -- 
root@petalinux:~# i2cdetect -y -a 2
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- 5e -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 

```



# Linux图形界面的搭建参考材料



## 参考材料

`zcu102`的`base_trd`

<https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842571/Zynq+UltraScale+MPSoC+Base+TRD>

<https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/520618122/Zynq%2BUltraScale%2BMPSoC%2BBase%2BTRD%2B2020.1#ZynqUltraScale+MPSoCBaseTRD2020.1-Software>

`zcu106`的`vcu_trd`
<https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/2428928001/Zynq+UltraScale+MPSoC+VCU+TRD+2022.2#2.1-TRD-Support>

其他的

<https://xilinx.github.io/vck190-base-trd/2020.2/html/build-plnx.html>



实际上, 根据后面展开的`device-tree`有关文件, `base_trd`更有参考价值



下载 `VCU TRD 2022.2 release package`. 大约有 4G

[rdf0428-zcu106-vcu-trd-2022-2.zip](https://www.xilinx.com/member/forms/download/xef.html?filename=rdf0428-zcu106-vcu-trd-2022-2.zip)



下载`Base TRD 2022.2 release package`, 大约有 4.6G

 [rdf0421-zcu102-base-trd-2020-1.zip](https://www.xilinx.com/member/forms/download/design-license-xef.html?filename=rdf0421-zcu102-base-trd-2020-1.zip)

<https://www.xilinx.com/products/board-docs/ek-u1-zcu102-g-docs.html>



## `base_trd`的`vivado`工程

```bash
$ export TRD_HOME=/home/andy/workdir/trd/rdf0421-zcu102-base-trd-2020-1
```

<https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/524189697/Zynq+UltraScale+MPSoC+Base+TRD+2020.1+-+Design+Module+6>

```bash
$ mkdir -p $TRD_HOME/vivado
$ cd $TRD_HOME/vivado
$ vivado
```

```tcl
% open_hw_platform ../zcu102_base_trd/hw/zcu102_base_trd.xsa
```

恢复的vivado工程位于

```bash
rdf0421-zcu102-base-trd-2020-1/vivado/zcu102_base_trd/prj
```

## `base_trd`的`petalinux`工程

```bash
$ $TRD_HOME/petalinux/sdk.sh -y -d $TRD_HOME/petalinux/sdk
PetaLinux SDK installer version 2020.1
======================================
You are about to install the SDK to "/home/andy/workdir/trd/rdf0421-zcu102-base-trd-2020-1/petalinux/sdk". Proceed [Y/n]? Y
Extracting SDK...................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................done
Setting it up...done
Your environment is misconfigured, you probably need to 'unset LD_LIBRARY_PATH'
but please check why this was set in the first place and that it's safe to unset.
The SDK will not operate correctly in most cases when LD_LIBRARY_PATH is set.
For more references see:
  http://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html#AEN80
  http://xahlee.info/UnixResource_dir/_/ldpath.html
/home/andy/workdir/trd/rdf0421-zcu102-base-trd-2020-1/petalinux/sdk/post-relocate-setup.sh: Failed to source /home/andy/workdir/trd/rdf0421-zcu102-base-trd-2020-1/petalinux/sdk/environment-setup-aarch64-xilinx-linux with status 1
SDK has been successfully set up and is ready to be used.
Each time you wish to use the SDK in a new shell session, you need to source the environment setup script e.g.
 $ . /home/andy/workdir/trd/rdf0421-zcu102-base-trd-2020-1/petalinux/sdk/environment-setup-aarch64-xilinx-linux
```

```bash
$ cd $TRD_HOME/petalinux

$ source /opt/Xilinx/PetaLinux/2020.1/tool/settings.sh 
PetaLinux environment set to '/opt-shadow/Xilinx/PetaLinux/2020.1/tool'
WARNING: This is not a supported OS
INFO: Checking free disk space
INFO: Checking installed tools
INFO: Checking installed development libraries
INFO: Checking network and other services

$ petalinux-create -t project -s zcu102-prod-base-dm10.bsp -n bsp
INFO: Create project: bsp
INFO: New project successfully created in /home/andy/workdir/trd/rdf0421-zcu102-base-trd-2020-1/petalinux/bsp

$ cd bsp/

$ petalinux-config --silentconfig	# 这里错误无法解决
INFO: sourcing build tools
[INFO] generating Kconfig for project
[INFO] silentconfig project
ERROR: Failed to silentconfig project component 
ERROR: Failed to config project.

$ ls $TRD_HOME/petalinux/bsp/project-spec/meta-user/recipes-bsp/device-tree/files/
base_trd  pl-custom.dtsi    zcu102-base-dm10.dtsi  zcu102-base-dm4.dtsi  zcu102-base-dm6.dtsi  zcu102-base-dm8.dtsi
common    system-user.dtsi  zcu102-base-dm1.dtsi   zcu102-base-dm5.dtsi  zcu102-base-dm7.dtsi  zcu102-base-dm9.dtsi
https://xilinx.github.io/vck190-base-trd/2020.2/html/build-plnx.html
$ rm $TRD_HOME/petalinux/bsp/project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi

$ cp $TRD_HOME/petalinux/bsp/project-spec/meta-user/recipes-bsp/device-tree/files/zcu102-base-dm1.dtsi $TRD_HOME/petalinux/bsp/project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi

$ petalinux-build
```



那么就参考device-tree吧

比如`idt8t49n24x`配置

```bash
#ifdef PLATFORM_ZCU104
		/* idt8t49n241 i2c clock generator */
		idt8t49n24x: clock-generator@6c {
			status = "okay";
			compatible = "idt,idt8t49n24x";
			#clock-cells = <1>;
			reg = <0x6c>;

			/* input clock(s); the XTAL is hard-wired on the ZCU104 board */
			clocks = <&refhdmi>;
			clock-names = "input-xtal";

			settings = [
				09 50 00 60 67 c5 6c 01 03 00 31 00 01 40 00 01 40 00 74 04 00 74 04 77 6d 00 00 00 00 00 00 ff
				ff ff ff 01 3f 00 2e 00 0d 00 00 00 01 00 00 d0 08 00 00 00 00 00 08 00 00 00 00 00 00 44 44 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 e9 0a 2b 20 00 00 00 0f 00 00 00 0e 00 00 0e 00 00 00 27 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				e3 00 08 01 00 00 00 00 00 00 00 00 00 b0 00 00 00 0a 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
				00 00 00 00 85 00 00 9c 01 d4 02 71 07 00 00 00 00 83 00 10 02 08 8c
				];
		};
#endif /* PLATFORM_ZCU104 */
```

`DP159`的设置

```bash
		/* DP159 exposes a virtual CCF clock. Upon .set_rate(), it adapts its retiming/driving behaviour */
		dp159: hdmi-retimer@5e {
			status = "okay";
			compatible = "ti,dp159";
			reg = <0x5e>;
			#address-cells = <1>;
			#size-cells = <0>;
			#clock-cells = <0>;
		};
```



## `vcu-trd`的`vivado`工程

```bash
$ export TRD_HOME=/home/andy/workdir/trd/rdf0428-zcu106-vcu-trd-2022-2
$ cd $TRD_HOME/pl
$ vivado -source designs/zcu106_trd/project.tcl		 	# 要确保版本是 2022.2
```

预编译的`xsa`文件在`rdf0428-zcu106-vcu-trd-2022-2/pl/prebuild/zcu106_trd`



## `vcu-trd`的`petalinux`工程

<https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/2428928250/Zynq+UltraScale+MPSoC+VCU+TRD+2022.2+-+Run+and+Build+Flow#3.2-VCU-PetaLinux-BSP>

```bash
$ source /opt/Xilinx/PetaLinux/2022.2/tool/settings.sh
$ cd $TRD_HOME/apu/vcu_petalinux_bsp
$ petalinux-create -t project -s xilinx-vcu-zcu106-v2022.2-final.bsp
INFO: Create project: 
INFO: Projects: 
INFO: 	* xilinx-vcu-zcu106-v2022.2-final
INFO: Has been successfully installed to /home/andy/workdir/trd/rdf0428-zcu106-vcu-trd-2022-2/apu/vcu_petalinux_bsp/
INFO: New project successfully created in /home/andy/workdir/trd/rdf0428-zcu106-vcu-trd-2022-2/apu/vcu_petalinux_bsp/

$ cd xilinx-vcu-zcu106-v2022.2-final

$ petalinux-config --silentconfig --get-hw-description=../../../pl/prebuild/zcu106_trd
==
$ petalinux-config --silentconfig --get-hw-description=$TRD_HOME/pl/prebuild/zcu106_trd

$ cd project-spec/meta-user/recipes-bsp/device-tree/files/
$ ln -sf vcu_trd.dtsi system-user.dtsi
$ cd ../../../../../

$ petalinux-build

$ petalinux-build --sdk
$ petalinux-package --sysroot

```





# `Video Frame Buffer`版本

一个问题, 用`VDMA`还是`Video Frame Buffer RW`?

后者可以设置视频流数据格式，适合在 Linux 下使用.

`dp159`的`oe`和`pll`的`rst`, `zcu106_vcu_trd`是对`oe`直接给高电平, `aresetn`直接给`rst`. 暂时维持现在的用`xgpio`复位不变吧.

也就是说, 整体上, 仅对`vdma`版本做了最小修改. 用`vfb`替换`vdma`, 另外`hls_ip`需要单独`reset`, 就这亮点修改.

`vitis`这边裸机验证, `vfb`这边的也好添加, 目的是验证`xsa`可用,就不加入`bsp`了.

##  `petalinux`碰到的问题, label错误或重复

`components/plnx_workspace/device-tree/device-tree/pl.dtsi` 里面有

```bash

		mipi_csi2_rx_axis_switch_0: axis_switch@80060000 {
			clock-names = "aclk", "s_axi_ctrl_aclk";
			clocks = <&zynqmp_clk 74>, <&zynqmp_clk 71>;
			compatible = "xlnx,axis-switch-1.1";
			reg = <0x0 0x80060000 0x0 0x10000>;
			xlnx,num-mi-slots = <1>;
			xlnx,num-si-slots = <2>;
			xlnx,routing-mode = <1>;
			axis_switch_portsmipi_csi2_rx_axis_switch_0: ports {
				#address-cells = <1>;
				#size-cells = <0>;
				axis_switch_port1mipi_csi2_rx_axis_switch_0: port@1 {
					reg = <1>;
					axis_switch_out1mipi_csi2_rx_axis_switch_0: endpoint {
						remote-endpoint = <&v_frmbuf_wr_0mipi_csi2_rx_axis_switch_0>;
					};
				};
				axis_switch_port0mipi_csi2_rx_axis_switch_0: port@0 {
					reg = <0>;
				};
			};
		};
```

我想在 `project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi` 覆盖定义
```bash
&mipi_csi2_rx_axis_switch_0 {
    ports {
        #address-cells = <1>;
        #size-cells = <0>;

        /* port@0 用作 v_proc_ss 的 remote-endpoint 目标（根据 pl.dtsi 的引用） */
        port@0 {
            reg = <0>;
            mipi_csi2_rx_axis_switch_0mipi_csi2_rx_v_proc_ss_0: endpoint {
            };
        };

        /* port@1 用作 v_tpg 的 remote-endpoint 目标 */
        port@1 {
            reg = <1>;
            mipi_csi2_rx_axis_switch_0mipi_csi2_rx_v_tpg_0: endpoint {
            };
        };
	
	port@2 {
            reg = <2>;
            axis_switch_out1mipi_csi2_rx_axis_switch_0: endpoint {
	        remote-endpoint = <&v_frmbuf_wr_0mipi_csi2_rx_axis_switch_0>;
            };
        };
    };
};
```

出现报错

```bash
Subprocess output:
/home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_peta/petalinux/build/tmp/work/zynqmp_generic-xilinx-linux/device-tree/xilinx-v2022.2+gitAUTOINC+24d29888d0-r0/system-user.dtsi:63.66-65.15: ERROR (duplicate_label): /amba_pl@0/axis_switch@80060000/ports/port@2/endpoint: Duplicate label 'axis_switch_out1mipi_csi2_rx_axis_switch_0' on /amba_pl@0/axis_switch@80060000/ports/port@2/endpoint and /amba_pl@0/axis_switch@80060000/ports/port@1/endpoint

```

### 问题的处理 

用 /delete-node/ 删除原节点再重建

```bash
&mipi_csi2_rx_axis_switch_0 {
    ports {
        #address-cells = <1>;
        #size-cells = <0>;

        /delete-node/ port@1;
        /delete-node/ port@0;
        
        /* port@0 用作 v_proc_ss 的 remote-endpoint 目标（根据 pl.dtsi 的引用） */
        port@0 {
            reg = <0>;
            mipi_csi2_rx_axis_switch_0mipi_csi2_rx_v_proc_ss_0: endpoint {
            };
        };

        /* port@1 用作 v_tpg 的 remote-endpoint 目标 */
        port@1 {
            reg = <1>;
            mipi_csi2_rx_axis_switch_0mipi_csi2_rx_v_tpg_0: endpoint {
            };
        };
	
	port@2 {
            reg = <2>;
            axis_switch_out1mipi_csi2_rx_axis_switch_0: endpoint {
	        remote-endpoint = <&v_frmbuf_wr_0mipi_csi2_rx_axis_switch_0>;
            };
        };
    };
};

```

# `IDT8T49N24X` 驱动及`dtg`配置

`IDT8T49N24X`驱动默认是包含的

```bash
$ petalinux-config -c kernel
-> Device Drivers                                                                                                               -> Common Clock Framework 
```



`reset-gpios`

```bash
reset-gpios = <&rest_gpio 0 1>;
reset-delay-us = <10000>;        // 延迟10ms

reset-gpios = <&rest_gpio 0 1>;
reset-delay-ms = <10>;        // 延迟10ms

<&GPIO控制器节点 GPIO编号 GPIO标志>

GPIO flags 常见数值
值	宏定义	解释
0	GPIO_ACTIVE_HIGH	高电平有效
1	GPIO_ACTIVE_LOW	低电平有效
2	GPIO_OPEN_DRAIN	开漏
3	GPIO_OPEN_SOURCE	开源
```

总线1设置

```bash
&axi_iic_1 {
    /* idt8t49n241 i2c clock generator */
    idt8t49n24x: clock-generator@7c {
        status = "okay";
        compatible = "idt,idt8t49n24x";
        #clock-cells = <1>;
        reg = <0x7c>;
        reset-gpios = <&rest_gpio 0 1>;
        reset-delay-ms = <200>;

        /* input clock(s); the XTAL is hard-wired on the board */
        clocks = <&refhdmi>;
        clock-names = "input-xtal";

        settings = [
            09 50 00 60 67 c5 6c 01 03 00 31 00 01 40 00 01 40 00 74 04 00 74 04 77 6d 00 00 00 00 00 00 ff
            ff ff ff 01 3f 00 2e 00 0d 00 00 00 01 00 00 d0 08 00 00 00 00 00 08 00 00 00 00 00 00 44 44 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 e9 0a 2b 20 00 00 00 0f 00 00 00 0e 00 00 0e 00 00 00 27 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            e3 00 08 01 00 00 00 00 00 00 00 00 00 b0 00 00 00 0a 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 85 00 00 9c 01 d4 02 71 07 00 00 00 00 83 00 10 02 08 8c
            ];
    };
};
```

## 问题: 看起来复位没有起作用

```bash
root@petalinux:~# dmesg | grep idt
[    4.924305] idt8t49n24x 1-007c: idt24x_probe
[    5.025710] idt8t49n24x 1-007c: error writing all settings to chip (-5)
[    5.040539] idt8t49n24x: probe of 1-007c failed with error -5
```

进入板子的系统后有
```bash
root@petalinux:~# dmesg | grep idt
[    4.924305] idt8t49n24x 1-007c: idt24x_probe
[    5.025710] idt8t49n24x 1-007c: error writing all settings to chip (-5)
[    5.040539] idt8t49n24x: probe of 1-007c failed with error -5
root@petalinux:~# dmesg | grep xgpio 
root@petalinux:~# dmesg | grep reset
root@petalinux:~# grep -R "800b0000" -n /proc/device-tree
/proc/device-tree/__symbols__/rest_gpio:1:/amba_pl@0/gpio@800b0000
root@petalinux:~# cat /sys/kernel/debug/gpio
gpiochip2: GPIOs 302-475, parent: platform/ff0a0000.gpio, zynqmp_gpio:

gpiochip1: GPIOs 476-507, parent: platform/800b0000.gpio, 800b0000.gpio:
 gpio-479 (                    |reset               ) out lo 
 gpio-480 (                    |reset               ) out hi ACTIVE LOW
 gpio-483 (                    |reset               ) out hi ACTIVE LOW
 gpio-484 (                    |reset               ) out hi ACTIVE LOW

gpiochip0: GPIOs 508-511, parent: platform/firmware:zynqmp-firmware:gpio, firmware:zynqmp-firmware:gpio:
 gpio-509 (                    |reset               ) out hi ACTIVE LOW


```


sysfs操作后可以访问
```bash
echo 476 > /sys/class/gpio/export
echo 0 > /sys/class/gpio/gpio476/value 
echo 1 > /sys/class/gpio/gpio476/value
i2cdetect -y -a 1

     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- -- -- -- -- -- 7c -- -- -- 
```


确实有驱动
```bash
root@petalinux:~# grep -R "idt8t49n24x" -n /lib/modules/*              
/lib/modules/5.15.36-xilinx-v2022.2/modules.builtin.modinfo:1:sha1_ce.alias=crypto-sha1sha1_ce.alias=sha1sha1_ce.license=GPL v2sha1_ce.file=arch/arm64/crypto/sha1-cesha1_ce.author=Ard Biesheuvel <ard.biesheuvel.
� opyareacfbcopyareaillrectcfbfillrectmgbltcfbimgbltci�z����cfi_cmdset_0001cfi_cmdset_0001013@�@00cfi_cmdset_0001mdset_002��(robecfi_probetilcfi_util_cu�C�Y�pbi ʠ�neonchacha_neon0__��chacha_neon                *
/lib/modules/5.15.36-xilinx-v2022.2/modules.builtin:156:kernel/drivers/clk/idt/clk-idt8t49n24x.ko
```

```bash
root@petalinux:~# dtc -I fs /proc/device-tree/amba_pl@0/i2c@80020000/clock-generator@7c -O dts                                                                                                                  
<stdout>: ERROR (name_properties): /: "name" property is incorrect ("clock-generator" instead of base node name)
ERROR: Input tree has errors, aborting (use -f to force output)
root@petalinux:~# ls /proc/device-tree/amba_pl@0/i2c@80020000/clock-generator@7c
#clock-cells    clock-names     clocks          compatible      name            phandle         reg             reset-delay-ms  reset-gpios     settings        status

root@petalinux:~# dtc -I fs /proc/device-tree -O dts | grep -n "clock-generator@7c"
```

### 解出源码

```bash
$ petalinux-devtool modify linux-xlnx
```

查看
```bash
petalinux/components/yocto/workspace/sources/linux-xlnx/drivers/clk/idt/clk-idt8t49n24x.c
```
没有任何关于 GPIO 或 reset 的调用，例如没有 gpiod_get() / devm_gpiod_get_optional()、没有 gpiod_set_value() / gpiod_set_value_cansleep()、也没有 gpio_request() / gpio_direction_output() 等。

也没有使用通用的 reset API（reset_control_get() / reset_control_assert() 等）。

### 解决方案

1. 补丁实现

```c
/* 在 clk-idt8t49n24x-core.h 或此文件的 chip 结构里加入 */
struct clk_idt24x_chip {
    ...
    struct gpio_desc *reset_gpiod; /* optional reset gpio */
    ...
};

```

在 `probe()` 中（在 `chip = devm_kzalloc(...)` 之后，regmap 初始化之前是合适的时机）加入：

```c
#include <linux/gpio/consumer.h>
#include <linux/delay.h>

    /* get optional reset gpio named "reset" in DT (reset-gpios) */
    chip->reset_gpiod = devm_gpiod_get_optional(&client->dev, "reset", GPIOD_OUT_HIGH);
    if (IS_ERR(chip->reset_gpiod)) {
        dev_err(&client->dev, "failed to get reset gpio\n");
        return PTR_ERR(chip->reset_gpiod);
    }

    /* If present, pulse reset: assert (active low) then deassert */
    if (chip->reset_gpiod) {
        /* adjust depending on wiring: if active_low, gpiod_get_optional with GPIOD_OUT_HIGH
           has set it to inactive state. Here we produce a reset pulse: */
        gpiod_set_value_cansleep(chip->reset_gpiod, 0); /* assert reset */
        msleep(20);
        gpiod_set_value_cansleep(chip->reset_gpiod, 1); /* deassert reset */
        msleep(50); /* wait for device to be ready */
    }

```



2. 仿照`vcu_trd`用`aresetn`复位

好处是不用给内核源码打补丁. 打补丁也需要进行测试, 太费事.



### 问题的处理

选择方案2, 不去修改内核源码为补丁.

新的`xsa`文件在`vitis`裸机测试`ok`.

```bash
rm -rf components/
rm -rf images/
petalinux-config --silentconfig --get-hw-description=../hardware/
petalinux-build -x mrproper
petalinux-build
petalinux-package --boot --u-boot --fpga --force
```



这次

```bash
root@petalinux:~# dmesg | grep idt
[    4.927039] idt8t49n24x 1-007c: idt24x_probe
[    5.069060] idt8t49n24x 1-007c: error writing all settings to chip (-5)
[    5.075718] idt8t49n24x: probe of 1-007c failed with error -5
root@petalinux:~# i2cdetect -y -a 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- 6c -- -- -- 
70: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 

```

艹, 从`7c`修改为`6c`, 再来

这次

```bash
root@petalinux:~# dmesg | grep idt
[    4.927039] idt8t49n24x 1-007c: idt24x_probe
[    5.069060] idt8t49n24x 1-007c: error writing all settings to chip (-5)
[    5.075718] idt8t49n24x: probe of 1-007c failed with error -5

root@petalinux:~# i2cdetect -y -a 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- -- -- -- -- -- 7c -- -- -- 
```

也就是要避免`0006`寄存器被修改为`6c`,保持`7c`

那么总线1设置

```bash
&axi_iic_1 {
    /* idt8t49n241 i2c clock generator */
    idt8t49n24x: clock-generator@7c {
        status = "okay";
        compatible = "idt,idt8t49n24x";
        #clock-cells = <1>;
        reg = <0x7c>;

        /* input clock(s); the XTAL is hard-wired on the board */
        clocks = <&refhdmi>;
        clock-names = "input-xtal";

        settings = [
            09 50 00 60 67 c5 7c 01 03 00 31 00 01 40 00 01 40 00 74 04 00 74 04 77 6d 00 00 00 00 00 00 ff
            ff ff ff 01 3f 00 2e 00 0d 00 00 00 01 00 00 d0 08 00 00 00 00 00 08 00 00 00 00 00 00 44 44 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 e9 0a 2b 20 00 00 00 0f 00 00 00 0e 00 00 0e 00 00 00 27 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            e3 00 08 01 00 00 00 00 00 00 00 00 00 b0 00 00 00 0a 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 85 00 00 9c 01 d4 02 71 07 00 00 00 00 83 00 10 02 08 8c
            ];
    };
};

```



这次

```bash
root@petalinux:~# dmesg | grep idt
[    4.928271] idt8t49n24x 1-007c: idt24x_probe
[    5.142424] idt8t49n24x 1-007c: idt24x_read_from_hw: initial values read from chip successfully
[    5.152024] idt8t49n24x 1-007c: probe success. input freq: 40000000Hz (XTAL), settings string? true

```

对了, lock灯也亮了.



# `vid_phy`的`dtg`配置

参考材料

<rdf0428-zcu106-vcu-trd-2022-2/apu/vcu_petalinux_bsp/xilinx-vcu-zcu106-v2022.2-final/project-spec/meta-user/recipes-bsp/device-tree/files>

<https://github.com/Xilinx/hdmi-modules/blob/xlnx_rel_v2022.1/Documentation/devicetree/bindings/xlnx%2Cvphy.txt>

<https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18841797/Xilinx+Phy+VideoPhy+Driver>

<https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/124747853/Understanding+clock+connection+in+Video+PHY+Device+Tree>



目前lock灯亮的, 说明`IDT8T49N24X`配置输出了一个时钟, 应该是`148.5MHz`,  但是现在`clk_cnt_0`计数为0. 接下来, 应配置`vid_phy_controller_0`节点使`clk_cnt_0`计数为2970000左右.



其实可以这样

```bash
&vid_phy_controller_0 {
    /delete-property/ clock-names;
    /delete-property/ clocks;
    
    clock-names = "mgtrefclk1_pad_p_in", "mgtrefclk1_pad_n_in", "vid_phy_tx_axi4s_aclk", "vid_phy_sb_aclk", "vid_phy_axi4lite_aclk", "drpclk";
    clocks = <&misc_clk_0>, <&misc_clk_0>, <&misc_clk_1>, <&zynqmp_clk 71>, <&zynqmp_clk 71>, <&zynqmp_clk 71>;
};
```

其中, `misc_clk_0`是`148500000`, `misc_clk_1`是`297000000`,`ynqmp_clk`的71口是`PL0_REF`

在 <components/plnx_workspace/device-tree/device-tree/include/dt-bindings/clock/xlnx-zynqmp-clk.h> 定义的

参考<components/plnx_workspace/device-tree/device-tree/zynqmp-clk-ccf.dtsi>



然后还是`clk_cnt_0`计数为0. 那么检查`idt8t49n24x`的配置序列, 初始化序列好像没有提供输出频率

<https://adaptivesupport.amd.com/s/question/0D52E00006hpkjJSAQ/idt8t49n241-linux-driver?language=en_US>

```bash
root@petalinux:~# cd /sys/kernel/debug/idt24x

root@petalinux:/sys/kernel/debug/idt24x# echo 148500000 > q2
root@petalinux:/sys/kernel/debug/idt24x# echo 1 > action
[ 1456.957854] idt8t49n24x 1-007c: idt24x_set_rate. calling idt24x_set_frequency for Q2. rate: 148500000

```

可在系统动态修改. 



<https://github.com/Xilinx/linux-xlnx/blob/master/Documentation/devicetree/bindings/clock/idt%2Cidt8t49n24x.txt>



```bash
&vid_phy_controller_0 {
    /delete-property/ clock-names;
    /delete-property/ clocks;

    clock-names = "mgtrefclk1_pad_p_in",
                  "mgtrefclk1_pad_n_in",
                  "vid_phy_tx_axi4s_aclk",
                  "vid_phy_sb_aclk",
                  "vid_phy_axi4lite_aclk",
                  "drpclk";

    clocks = <&idt8t49n24x 2>, <&idt8t49n24x 2>,
             <&misc_clk_1>,
             <&zynqmp_clk 71>, <&zynqmp_clk 71>, <&zynqmp_clk 71>;

    assigned-clocks = <&idt8t49n24x 2>;
    assigned-clock-rates = <148500000>;
};

```

实际上也没有在`ila`看到`clk_cnt`计数呢





基本上下面配置应该要有, 但是`vphy`没有运行, 所以没有检测到`297MHz`, 是否需要添加x-server之类的东西?

```bash
&amba {
    refhdmi: refhdmi {
        compatible = "fixed-clock";
        #clock-cells = <0>;
        clock-frequency = <40000000>;
    };
    hdmi_dru_clk: hdmi_dru_clk {
        compatible = "fixed-clock";
        #clock-cells = <0>;
        clock-frequency = <100000000>;
    };

    axi_lite_clk: axi_lite_clk {
        compatible = "fixed-factor-clock";
        clocks = <&zynqmp_clk 71>;
        #clock-cells = <0>;
        clock-div = <1>;
        clock-mult = <1>;
    };
    axi_stream_clk: axi_stream_clk {
        compatible = "fixed-factor-clock";
        clocks = <&zynqmp_clk 74>;
        #clock-cells = <0>;
        clock-div = <1>;
        clock-mult = <1>;
    };
};

&axi_iic_1 {
    /* idt8t49n241 i2c clock generator */
    idt8t49n24x: clock-generator@7c {
        status = "okay";
        compatible = "idt,idt8t49n24x";
        #clock-cells = <1>;
        reg = <0x7c>;

        /* input clock(s); the XTAL is hard-wired on the board */
        clocks = <&refhdmi>;
        clock-names = "input-xtal";

        settings = [
            09 50 00 60 67 c5 7c 01 03 00 31 00 01 40 00 01 40 00 74 04 00 74 04 77 6d 00 00 00 00 00 00 ff
            ff ff ff 01 3f 00 2e 00 0d 00 00 00 01 00 00 d0 08 00 00 00 00 00 08 00 00 00 00 00 00 44 44 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 e9 0a 2b 20 00 00 00 0f 00 00 00 0e 00 00 0e 00 00 00 27 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            e3 00 08 01 00 00 00 00 00 00 00 00 00 b0 00 00 00 0a 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
            00 00 00 00 85 00 00 9c 01 d4 02 71 07 00 00 00 00 83 00 10 02 08 8c
            ];
    };
};

&axi_iic_2 {
    /* DP159 exposes a virtual CCF clock. Upon .set_rate(), it adapts its retiming/driving behaviour */
    dp159: hdmi-retimer@5e {
        status = "okay";
        compatible = "ti,dp159";
        reg = <0x5e>;
        #address-cells = <1>;
        #size-cells = <0>;
        #clock-cells = <0>;
    };
};


&vid_phy_controller_0 {
    clocks = <&axi_lite_clk>, <&hdmi_dru_clk>;
    clock-names = "vid_phy_axi4lite_aclk", "dru-clk";
};

&v_hdmi_tx_ss_0{
    clocks = <&axi_lite_clk>, <&axi_stream_clk>, <&idt8t49n24x 2>, <&dp159>;
    clock-names = "s_axi_cpu_aclk", "s_axis_video_aclk", "txref-clk", "retimer-clk";
};


```

# 尝试借鉴base_trd和vcu_trd

从`rdf0421-zcu102-base-trd-2020-1/petalinux/bsp`复制和添加文件和文件夹, 主要是

`project-spec/configs/rootfs_config`

```bash
CONFIG_openssh-sftp-server=y
CONFIG_imageclass-populate-sdk-qt5=y
CONFIG_packagegroup-trd=y
```

`project-spec/meta-user/conf/user-rootfsconfig`

```bash
CONFIG_packagegroup-trd
```



`project-spec/meta-user/recipes-kernel/linux/linux-xlnx/bsp.cfg`

`project-spec/meta-user/recipes-apps`

报错

```bash
$ petalinux-build
[INFO] Sourcing buildtools
[INFO] Building project
[INFO] Sourcing build environment
[INFO] Generating workspace directory
INFO: bitbake petalinux-image-minimal
NOTE: Started PRServer with DBfile: /home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_peta/petalinux/build/cache/prserv.sqlite3, Address: 127.0.0.1:43417, PID: 1633412
WARNING: Host distribution "ubuntu-22.04" has not been validated with this version of the build system; you may possibly experience unexpected failures. It is recommended that you use a tested distribution.
Loading cache: 100% |###############################################################################################################################################################################| Time: 0:00:00
Loaded 6355 entries from dependency cache.
Parsing recipes: 100% |#############################################################################################################################################################################| Time: 0:00:03
Parsing of 4466 .bb files complete (4373 cached, 93 parsed). 6502 targets, 577 skipped, 1 masked, 0 errors.
NOTE: Resolving any missing task queue dependencies
ERROR: Nothing RPROVIDES 'packagegroup-trd' (but /home/andy/workdir/zirui/04_hdmi_tx/hdmi_tx_peta/petalinux/components/yocto/layers/meta-petalinux/recipes-core/images/petalinux-image-minimal.bb RDEPENDS on or otherwise requires it)
NOTE: Runtime target 'packagegroup-trd' is unbuildable, removing...
Missing or unbuildable dependency chain was: ['packagegroup-trd']
NOTE: Runtime target 'kernel-modules' is unbuildable, removing...
Missing or unbuildable dependency chain was: ['kernel-modules', 'petalinux-image-minimal', 'packagegroup-trd']
ERROR: Required build target 'petalinux-image-minimal' has no buildable providers.
Missing or unbuildable dependency chain was: ['petalinux-image-minimal', 'packagegroup-trd']


```

`packagegroup-trd`没有了. 很可能是在`components/yocto`定义的, 而本机对于`base_trd`工程的`petalinux-config --silentconfig`这一步失败, 所以没法查看

解决办法, 在`ubuntu18.04`的虚拟机里安装2020.1版本的`petalinux`再导入

```bash
petalinux-config --silentconfig
INFO: sourcing build tools
[INFO] generating Kconfig for project
[INFO] silentconfig project
[INFO] extracting yocto SDK to components/yocto
[INFO] sourcing build environment
[INFO] generating kconfig for Rootfs
[INFO] silentconfig rootfs
[INFO] generating plnxtool conf
[INFO] generating user layers
[INFO] generating workspace directory
[INFO] successfully configured project

```

就导入成功了

`grep -R "packagegroup"`

```bash
project-spec/meta-user/recipes-core/packagegroups/packagegroup-trd.bb:inherit packagegroup
project-spec/meta-user/recipes-core/packagegroups/packagegroup-trd.bb:	packagegroup-core-tools-debug \
project-spec/meta-user/recipes-core/packagegroups/packagegroup-trd.bb:	packagegroup-petalinux-display-debug \
project-spec/meta-user/recipes-core/packagegroups/packagegroup-trd.bb:	packagegroup-petalinux-gstreamer \
project-spec/meta-user/recipes-core/packagegroups/packagegroup-trd.bb:	packagegroup-petalinux-openamp \
project-spec/meta-user/recipes-core/packagegroups/packagegroup-trd.bb:	packagegroup-petalinux-opencv \
project-spec/meta-user/recipes-core/packagegroups/packagegroup-trd.bb:	packagegroup-petalinux-qt \
project-spec/meta-user/recipes-core/packagegroups/packagegroup-trd.bb:	packagegroup-petalinux-v4lutils \
project-spec/meta-user/recipes-core/packagegroups/packagegroup-trd.bb:	packagegroup-petalinux-x11 \
```

其实在这定义的

然后

搜一下：

```bash
grep -R "_append" project-spec/meta-user
grep -R "_prepend" project-spec/meta-user
```

凡是看到：

```bash
xxx_append
xxx_prepend
```

**在新 `bitbake` 里都应该改成：**

```bash
xxx:append
xxx:prepend
```



## `modetest`检测不到设备

其实还是没有显示, `startx`也没有成功

```bash
root@petalinux:~# modetest       
trying to open device 'i915'...failed
trying to open device 'amdgpu'...failed
trying to open device 'radeon'...failed
trying to open device 'nouveau'...failed
trying to open device 'vmwgfx'...failed
trying to open device 'omapdrm'...failed
trying to open device 'exynos'...failed
trying to open device 'tilcdc'...failed
trying to open device 'msm'...failed
trying to open device 'sti'...failed
trying to open device 'tegra'...failed
trying to open device 'imx-drm'...failed
trying to open device 'rockchip'...failed
trying to open device 'atmel-hlcdc'...failed
trying to open device 'fsl-dcu-drm'...failed
trying to open device 'vc4'...failed
trying to open device 'virtio_gpu'...failed
trying to open device 'mediatek'...failed
trying to open device 'meson'...failed
trying to open device 'pl111'...failed
trying to open device 'stm'...failed
trying to open device 'sun4i-drm'...failed
trying to open device 'armada-drm'...failed
trying to open device 'komeda'...failed
trying to open device 'imx-dcss'...failed
trying to open device 'mxsfb-drm'...failed
no device found
```

`ls /dev/dri`为空

其他

```bash
modetest -M xlnx
modetest -D b00c0000.v_mix

ls -l /dev/dri
dmesg | grep -i drm
dmesg | grep -i xilinx
dmesg | grep -i mix
ls /sys/bus/platform/devices | grep v_mix
ls /sys/bus/platform/devices | grep v

TRD 默认 alias 里就有
alias modetest-hdmi='modetest -D b00c0000.v_mix'

```



```bash
root@petalinux:~# zcat /proc/config.gz | grep DRM_XLNX
CONFIG_DRM_XLNX=y
CONFIG_DRM_XLNX_BRIDGE=y
CONFIG_DRM_XLNX_BRIDGE_DEBUG_FS=y
CONFIG_DRM_XLNX_DPTX=y
CONFIG_DRM_XLNX_DSI=y
CONFIG_DRM_XLNX_HDMITX=y
CONFIG_DRM_XLNX_MIXER=y
CONFIG_DRM_XLNX_PL_DISP=y
CONFIG_DRM_XLNX_SDI=y
CONFIG_DRM_XLNX_BRIDGE_CSC=y
CONFIG_DRM_XLNX_BRIDGE_SCALER=y
CONFIG_DRM_XLNX_BRIDGE_VTC=y
root@petalinux:~# ls /sys/bus/platform/devices | grep v
80080000.v_demosaic
80090000.v_gamma_lut
800a0000.v_tpg
800c0000.v_proc_ss
80100000.v_frmbuf_rd
80110000.v_frmbuf_wr
80120000.v_hdmi_tx_ss
80140000.vid_phy_controller
firmware:zynqmp-firmware:nvmem_firmware
```



在`Xilinx DRM`架构里

```bash
v_mix   		= CRTC / Plane / Primary display, 显示核心（没有它就没显示设备）
v_hdmi_tx_ss  	= Display Encoder
vid_phy 		= PHY
```



接下来咋办? 

再试试加一个 v_mix, 不行再迁移简化`trd`工程

***

## 添加一个 v_mix

| git commit | xsa md5                          |
| ---------- | -------------------------------- |
| 620d919a   | 0b7d84be9839665e779817b3c02c6574 |

[620d919a/system.pdf](620d919a/system.pdf)

[620d919a/mipi_csi2_rx.pdf](620d919a/mipi_csi2_rx.pdf)

![Snipaste_2026-01-23_10-50-45](620d919a/Snipaste_2026-01-23_10-50-45.png)

![Snipaste_2026-01-23_10-38-14](620d919a/Snipaste_2026-01-23_10-38-14.png)

```bash
petalinux-config --silentconfig --get-hw-description=../hardware/
```

修改好`device-tree`和内核`boot`参数

```bash
petalinux-build
petalinux-package --boot --u-boot --fpga --force
```

这次启动之后

```bash
root@petalinux:~# dmesg | grep xlnx-drm
[   13.224122] xlnx-drm-hdmi 80120000.v_hdmi_tx_ss: probe started
[   13.230028] xlnx-drm-hdmi 80120000.v_hdmi_tx_ss: hdmi tx audio disabled in DT
[   13.245027] xlnx-drm-hdmi 80120000.v_hdmi_tx_ss: probe successful
[   13.280032] xlnx-drm xlnx-drm.0: bound 80140000.v_mix (ops 0xffff800008e68fd0)
[   13.287309] xlnx-drm xlnx-drm.0: bound 80120000.v_hdmi_tx_ss (ops xlnx_drm_hdmi_component_ops [xilinx_hdmi_tx])
root@petalinux:~# ls /dev/dri
by-path  card0
root@petalinux:~# modetest -D 80140000.v_mix 45:3840x2160-60@BG24
trying to open device 'i915'...done
Encoders:
id      crtc    type    possible crtcs  possible clones
37      0       TMDS    0x00000001      0x00000001

Connectors:
id      encoder status          name            size (mm)       modes   encoders
38      0       connected       HDMI-A-1        480x270         28      37
  modes:
        index name refresh (Hz) hdisp hss hse htot vdisp vss vse vtot
  #0 3840x2160 59.85 3840 3888 3920 4010 2160 2163 2168 2222 533250 flags: phsync, pvsync; type: preferred, driver
  #1 3840x2160 60.00 3840 4016 4104 4400 2160 2168 2178 2250 594000 flags: phsync, pvsync; type: driver
  #2 3840x2160 59.94 3840 4016 4104 4400 2160 2168 2178 2250 593407 flags: phsync, pvsync; type: driver
  #3 3840x2160 30.00 3840 4016 4104 4400 2160 2168 2178 2250 297000 flags: phsync, pvsync; type: driver
  #4 3840x2160 29.97 3840 4016 4104 4400 2160 2168 2178 2250 296703 flags: phsync, pvsync; type: driver
  #5 3840x2160 29.98 3840 3888 3920 4000 2160 2163 2168 2191 262750 flags: phsync, nvsync; type: driver
  #6 2560x1440 59.95 2560 2608 2640 2720 1440 1443 1448 1481 241500 flags: phsync, nvsync; type: driver
  #7 1920x1080 60.00 1920 2008 2052 2200 1080 1084 1089 1125 148500 flags: phsync, nvsync; type: driver
  #8 1920x1080 60.00 1920 2008 2052 2200 1080 1084 1089 1125 148500 flags: phsync, pvsync; type: driver
  #9 1920x1080 59.94 1920 2008 2052 2200 1080 1084 1089 1125 148352 flags: phsync, pvsync; type: driver
  #10 1920x1080 50.00 1920 2448 2492 2640 1080 1084 1089 1125 148500 flags: phsync, pvsync; type: driver
  #11 1680x1050 59.88 1680 1728 1760 1840 1050 1053 1059 1080 119000 flags: phsync, nvsync; type: driver
  #12 1600x900 60.00 1600 1624 1704 1800 900 901 904 1000 108000 flags: phsync, pvsync; type: driver
  #13 1280x1024 60.02 1280 1328 1440 1688 1024 1025 1028 1066 108000 flags: phsync, pvsync; type: driver
  #14 1440x900 59.90 1440 1488 1520 1600 900 903 909 926 88750 flags: phsync, nvsync; type: driver
  #15 1280x960 60.00 1280 1376 1488 1800 960 961 964 1000 108000 flags: phsync, pvsync; type: driver
  #16 1366x768 59.79 1366 1436 1579 1792 768 771 774 798 85500 flags: phsync, pvsync; type: driver
  #17 1280x800 59.91 1280 1328 1360 1440 800 803 809 823 71000 flags: phsync, nvsync; type: driver
  #18 1280x720 60.00 1280 1390 1430 1650 720 725 730 750 74250 flags: phsync, pvsync; type: driver
  #19 1280x720 59.94 1280 1390 1430 1650 720 725 730 750 74176 flags: phsync, pvsync; type: driver
  #20 1280x720 50.00 1280 1720 1760 1980 720 725 730 750 74250 flags: phsync, pvsync; type: driver
  #21 1024x768 60.00 1024 1048 1184 1344 768 771 777 806 65000 flags: nhsync, nvsync; type: driver
  #22 800x600 60.32 800 840 968 1056 600 601 605 628 40000 flags: phsync, pvsync; type: driver
  #23 720x576 50.00 720 732 796 864 576 581 586 625 27000 flags: nhsync, nvsync; type: driver
  #24 720x480 60.00 720 736 798 858 480 489 495 525 27027 flags: nhsync, nvsync; type: driver
  #25 720x480 59.94 720 736 798 858 480 489 495 525 27000 flags: nhsync, nvsync; type: driver
  #26 640x480 60.00 640 656 752 800 480 490 492 525 25200 flags: nhsync, nvsync; type: driver
  #27 640x480 59.94 640 656 752 800 480 490 492 525 25175 flags: nhsync, nvsync; type: driver
  props:
        1 EDID:
                flags: immutable blob
                blobs:

                value:
                        00ffffffffffff00212b010031303030
                        1f18010380301b782aa220a656489a24
                        12505421080081c0814081809500b300
                        a9c0810001014dd000aaf0703e803020
                        3500dd0c1100001e023a801871382d40
                        582c4500dd0c1100001a000000fc004d
                        585830380a20202020202020000000ff
                        004150574232484130323533303001d3
                        020328b14b9004031f13011202115f61
                        6b030c001000883c2000200167d85dc4
                        01788003e30f0006a36600a0f0701f80
                        302035000f282100001a662156aa5100
                        1e30468f33000f282100001e565e00a0
                        a0a02950302035000f282100001a0000
                        00000000000000000000000000000000
                        000000000000000000000000000000fe
        2 DPMS:
                flags: enum
                enums: On=0 Standby=1 Suspend=2 Off=3
                value: 3
        5 link-status:
                flags: enum
                enums: Good=0 Bad=1
                value: 0
        6 non-desktop:
                flags: immutable range
                values: 0 1
                value: 0
        4 TILE:
                flags: immutable blob
                blobs:

                value:
        8 GEN_HDR_OUTPUT_METADATA:
                flags: blob
                blobs:

                value:
        39 colorspace:
                flags: range
                values: 0 12
                value: 0
        40 ycbcr_enc:
                flags: range
                values: 0 8
                value: 0
        41 xfer_func:
                flags: range
                values: 0 7
                value: 0
        42 quantization:
                flags: range
                values: 0 2
                value: 0
        43 height_out:
                flags: range
                values: 480 2160
                value: 0
        44 width_out:
                flags: range
                values: 640 4096
                value: 0
        45 in_fmt:
                flags: range
                values: 4106 8448
                value: 0
        46 out_fmt:
                flags: range
                values: 4106 8448
                value: 0
        47 aspect_ratio:
                flags: range
                values: 0 3
                value: 0

CRTCs:
id      fb      pos     size
36      0       (0,0)   (0x0)
  #0  nan 0 0 0 0 0 0 0 0 0 flags: ; type: 
  props:
        25 VRR_ENABLED:
                flags: range
                values: 0 1
                value: 0

Planes:
id      crtc    fb      CRTC x,y        x,y     gamma size      possible crtcs
34      0       0       0,0             0,0     0               0x00000001
  formats: RG24
  props:
        9 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 2
        32 scale:
                flags: range
                values: 0 2
                value: 4294934528
        33 alpha:
                flags: range
                values: 0 256
                value: 4294901768
35      0       0       0,0             0,0     0               0x00000001
  formats: BG24
  props:
        9 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 1

Frame buffers:
id      size    pitch

root@petalinux:~# modetest -M xlnx -p
CRTCs:
id      fb      pos     size
36      0       (0,0)   (3840x2160)
  #0 3840x2160 60.00 3840 4016 4104 4400 2160 2168 2178 2250 594000 flags: phsync, pvsync; type: driver
  props:
        25 VRR_ENABLED:
                flags: range
                values: 0 1
                value: 0

Planes:
id      crtc    fb      CRTC x,y        x,y     gamma size      possible crtcs
34      0       0       0,0             0,0     0               0x00000001
  formats: RG24
  props:
        9 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 2
        32 scale:
                flags: range
                values: 0 2
                value: 4294934528
        33 alpha:
                flags: range
                values: 0 256
                value: 4294901768
35      0       0       0,0             0,0     0               0x00000001
  formats: BG24
  props:
        9 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 1

```

### 查看显示能力的命令

```bash
modetest -M xlnx
modetest -M xlnx -p

如果知道v_mix的axi空间地址, 例如 0x80140000
modetest -D 80140000.v_mix  == modetest -M xlnx
```

主要是为了获得 `connect` 号, `crtc` 号 和 `plane`号, 以及支持的`modes`用于命令

比如下面的 `-s 41@39`的意思是 `connector_id@crtc_id`, `-P 38@39`的意思是`-P plane_id@crtc_id`

```bash
modetest -M xlnx -s 41@39:1920x1080-60 -P 38@39:1920x1080@BG24
modetest -D 80140000.v_mix -s 41@39:1920x1080-60 -P 38@39:1920x1080@BG24
```

### `modtest`参数

```bash
root@petalinux:~# modetest --help
usage: modetest [-acDdefMPpsCvrw]

 Query options:

        -c      list connectors
        -e      list encoders
        -f      list framebuffers
        -p      list CRTCs and planes (pipes)

 Test options:

        -P <plane_id>@<crtc_id>:<w>x<h>[+<x>+<y>][*<scale>][@<format>]  set a plane
        -s <connector_id>[,<connector_id>][@<crtc_id>]:[#<mode index>]<mode>[-<vrefresh>][@<format>]    set a mode
        -C      test hw cursor
        -v      test vsynced page flipping
        -r      set the preferred mode for all connectors
        -w <obj_id>:<prop_name>:<value> set property
        -a      use atomic API
        -F pattern1,pattern2    specify fill patterns

 Generic options:

        -d      drop master after mode set
        -M module       use the given driver
        -D device       use the given device

        Default is to dump all info.

```



没弄明白之前尝试了下面命令

```bash
modetest -M xlnx -s 30@28:1920x1080-60@YUYV
modetest -D 80140000.v_mix 45:3840x2160-60@BG24
modetest -M xlnx -s 30@28:1920x1080-60@YUYV
modetest -D 80140000.v_mix
modetest -D 80140000.v_mix 45:3840x2160-60@BG24
```

到这里`hdmi`接口是没有显示的



### 显示彩条

```bash
root@petalinux:~# modetest -M xlnx -s 38@36:3840x2160-60@BG24
setting mode 3840x2160-60.00Hz on connectors 38, crtc 36
[  237.248704] idt8t49n24x 1-007c: idt24x_set_rate. calling idt24x_set_frequency for Q2. rate: 148500000
```
结束彩条后显示纯色blue

```bash
root@petalinux:~# modetest -M xlnx -s 38@36:1920x1080-60@BG24
setting mode 1920x1080-60.00Hz on connectors 38, crtc 36
^C
root@petalinux:~# modetest -M xlnx -s 38@36:3840x2160-60@BG24
setting mode 3840x2160-60.00Hz on connectors 38, crtc 3
```
显示彩条

再次运行modetest -M xlnx -s 38@36:3840x2160-60@BG24之类命令会显示不规则彩条, ctrl+c结束命令之后显示纯蓝
```bash
modetest -M xlnx \
  -s 38@36:1920x1080-60@BG24 \
  -P 31@36:1920x1080+0+0@BG24
```

`startx`也报错没成功

```bash
root@petalinux:~# cat /var/log/Xorg.0.log | grep -E "(EE|drm|kms|plane)"
        (WW) warning, (EE) error, (NI) not implemented, (??) unknown.
[   972.361] (II) xfree86: Adding drm device (/dev/dri/card0)
[   972.361]    falling back to /sys/devices/platform/amba_pl@0/80140000.v_mix/drm/card0
[   972.362] (EE) Failed to load module "glx" (module does not exist, 0)
[   972.647] (EE) ARMSOC(0): ERROR: drm failed to set mode: Invalid argument
[   972.647] (EE) ARMSOC(0): ERROR: xf86SetDesiredModes() failed!
[   972.647] (EE) ARMSOC(0): ERROR: ARMSOCEnterVT() failed!
[   972.651] (EE) 
[   972.651] (EE) AddScreen/ScreenInit failed for driver 0
[   972.651] (EE) 
[   972.651] (EE) 
[   972.651] (EE) Please also check the log file at "/var/log/Xorg.0.log" for additional information.
[   972.651] (EE) 
[   972.903] (EE) Server terminated with error (1). Closing log file.
```



尝试用GStreamer输出彩条之类的图案


```bash
root@petalinux:~# gst-launch-1.0 \
>   videotestsrc pattern=smpte \
>   ! video/x-raw,format=BGRx,width=1920,height=1080,framerate=60/1 \
>   ! videoconvert \
>   ! kmssink driver-name=xlnx sync=false
Setting pipeline to PAUSED ...
Pipeline is PREROLLING ...
[ 1350.116394] [drm:xlnx_mix_plane_atomic_update] *ERROR* failed to mode-set a plane
Pipeline is PREROLLED ...
Setting pipeline to PLAYING ...
New clock: GstSystemClock
[ 1350.143330] [drm:xlnx_mix_plane_atomic_update] *ERROR* failed to mode-set a plane
[ 1350.211866] [drm:xlnx_mix_plane_atomic_update] *ERROR* failed to mode-set a plane
[ 1350.278506] [drm:xlnx_mix_plane_atomic_update] *ERROR* failed to mode-set a plane
[ 1350.345143] [drm:xlnx_mix_plane_atomic_update] *ERROR* failed to mode-set a plane
```
我的`v_mix_0`是设置成0个`overlay layers`的, 很可能是这个原因导致的

***


## 给`v_mix`添加一个`overlay layer`



| git commit | xsa md5                          |
| ---------- | -------------------------------- |
| 03076f8c   | 84661bbbe009421a1a8ec931f36c0057 |

[03076f8c/system.pdf](03076f8c/system.pdf)

[03076f8c/mipi_csi2_rx.pdf](03076f8c/mipi_csi2_rx.pdf)

![Snipaste_2026-01-23_10-59-46](03076f8c/Snipaste_2026-01-23_10-59-46.png)

![Snipaste_2026-01-23_11-01-15](03076f8c/Snipaste_2026-01-23_11-01-15.png)

避免报错, 在`vivado`的`bd`中添加之后

```tcl
reset_run synth_1
launch_runs synth_1 -jobs 8
```

确认添加成功

```bash
root@petalinux:~# modetest -M xlnx -p
CRTCs:
id      fb      pos     size
36      0       (0,0)   (0x0)
  #0  nan 0 0 0 0 0 0 0 0 0 flags: ; type: 
  props:
        25 VRR_ENABLED:
                flags: range
                values: 0 1
                value: 0

Planes:
id      crtc    fb      CRTC x,y        x,y     gamma size      possible crtcs
34      0       0       0,0             0,0     0               0x00000001
  formats: NV12
  props:
        9 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 0
35      0       0       0,0             0,0     0               0x00000001
  formats: BG24
  props:
        9 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 1

root@petalinux:~# modetest -M xlnx -s 38@36:3840x2160-60@BG24
setting mode 3840x2160-60.00Hz on connectors 38, crtc 36
[  292.962548] idt8t49n24x 1-007c: idt24x_set_rate. calling idt24x_set_frequency for Q2. rate: 148500000
^C
root@petalinux:~# modetest -M xlnx -p
CRTCs:
id      fb      pos     size
36      0       (0,0)   (3840x2160)
  #0 3840x2160 60.00 3840 4016 4104 4400 2160 2168 2178 2250 594000 flags: phsync, pvsync; type: driver
  props:
        25 VRR_ENABLED:
                flags: range
                values: 0 1
                value: 0

Planes:
id      crtc    fb      CRTC x,y        x,y     gamma size      possible crtcs
34      0       0       0,0             0,0     0               0x00000001
  formats: NV12
  props:
        9 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 0
35      0       0       0,0             0,0     0               0x00000001
  formats: BG24
  props:
        9 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 1

```

`modetest`和`GStreamer`输出彩条之类的图案, 成功

```bash
modetest -M xlnx -s 38@36:3840x2160-60@BG24

gst-launch-1.0 \
  videotestsrc pattern=snow \
  ! video/x-raw,format=BGRx,width=3840,height=2160,framerate=60/1 \
  ! videoconvert \
  ! kmssink driver-name=xlnx sync=false
```



### 用`GStreamer`输出彩条图案常用命令

```
gst-launch-1.0 \
  videotestsrc pattern=smpte \
  ! video/x-raw,format=BGRx,width=1920,height=1080,framerate=60/1 \
  ! videoconvert \
  ! kmssink driver-name=xlnx sync=false
  
gst-launch-1.0 \
  videotestsrc pattern=smpte \
  ! video/x-raw,format=BGRx,width=3840,height=2160,framerate=30/1 \
  ! videoconvert \
  ! kmssink driver-name=xlnx sync=false
  
gst-launch-1.0 \
  videotestsrc pattern=smpte \
  ! video/x-raw,format=BGRx,width=3840,height=2160,framerate=60/1 \
  ! videoconvert \
  ! kmssink driver-name=xlnx sync=false
  
gst-launch-1.0 \
  videotestsrc pattern=gradient \
  ! video/x-raw,format=BGRx,width=3840,height=2160,framerate=60/1 \
  ! videoconvert \
  ! kmssink driver-name=xlnx sync=false
 
```



#### Gstreamer Pattern

| pattern 名称        | 描述               |
| ------------------- | ------------------ |
| `smpte`             | 标准 SMPTE 彩条    |
| `snow`              | 雪花噪声（白噪声） |
| `black`             | 全黑画面           |
| `white`             | 全白画面           |
| `red`               | 红色填充           |
| `green`             | 绿色填充           |
| `blue`              | 蓝色填充           |
| `checkers-1`        | 黑白棋盘格         |
| `checkers-2`        | 彩色棋盘格         |
| `checkers-4`        | 4x4 彩色格         |
| `checkers-8`        | 8x8 彩色格         |
| `circular`          | 圆形渐变           |
| `blink`             | 黑白闪烁（交替）   |
| `gradient`          | 灰度渐变           |
| `ball`              | 滚动小球动画       |
| `pinwheel`          | 风车动画           |
| `zone-plate`        | 高频变化条纹       |
| `gamut`             | 色域测试图         |
| `chroma-zone-plate` | 彩色高频条纹       |



### `startx`依然失败

```bash
(EE) ARMSOC(0): ERROR: drm failed to set mode: Invalid argument
(EE) ARMSOC(0): ERROR: xf86SetDesiredModes() failed!

```

`X11` 要求 `KMS` 创建一个`primary plane + framebuffer`的组合, 但` xlnx-drm` 驱动没法满足它

**当前 DRM 资源结构**

```bash
Planes:
plane-0: NV12   type = Overlay
plane-1: BG24   type = Primary
```





### `tpg`显示没有成功

现在路径准备如下, 
```bash
[TPG] ---> [axis_switch] ---+
                             \
                              -> [v_framebuf_wr] -> PS DDR -> [v_framebuf_rd] -> [v_mix] -> HDMI
[CSI2-RX] -> [axis_switch] -/                                            PS DDR -/
```
现在`modetest -M xlnx -s 38@36:3840x2160-60@BG24`之后, 恢复蓝色背景显示之后

可以执行gst命令并显示在hdmi显示器上

```  bash
 gst-launch-1.0 \
  videotestsrc pattern=smpte \
  ! video/x-raw,format=BGRx,width=3840,height=2160,framerate=30/1 \
  ! videoconvert \
  ! kmssink driver-name=xlnx sync=false
```

后台执行`modetest -M xlnx -s 38@36:3840x2160-60@BG24&`, `cat /sys/kernel/debug/dri/0/state`才有`paddr`, 蓝屏状态没有

tpg_to_hdmi.sh
```bash
#!/bin/sh

# ===============================
# Base addresses
# ===============================
AXIS_SW=0x80060000
TPG=0x800A0000
FBWR=0x80110000

# DRM framebuffer physical address (from /sys/kernel/debug/dri/0/state)
FBADDR=0x37800000

# ===============================
# Stop everything first
# ===============================
echo "Stop TPG"
devmem $TPG 32 0

echo "Stop FBWR"
devmem $FBWR 32 0

# ===============================
# Configure TPG
# ===============================
echo "Configure TPG"

# height
devmem 0x800A0010 32 2160
# width
devmem 0x800A0018 32 3840
# pattern = color bars
devmem 0x800A0020 32 2
# color format = BGR
devmem 0x800A0028 32 2

# ===============================
# AXIS switch: select TPG (S1) -> M0
# ===============================
echo "Configure AXIS switch"

# M0 select S1
devmem 0x80060040 32 1
# enable switch
devmem 0x80060000 32 2

# ===============================
# Configure Framebuffer Writer
# ===============================
echo "Configure FBWR"

# buffer address
devmem 0x80110010 32 $((FBADDR))

# stride (bytes)
devmem 0x80110018 32 11520

# width
devmem 0x80110020 32 3840

# height
devmem 0x80110028 32 2160

# format BG24 (decimal value of 0x34324742)
devmem 0x80110030 32 875713346

# ===============================
# Start pipeline
# ===============================
#echo "Start FBWR"
#devmem 0x80110000 32 1
echo "Start FBWR (auto-restart)"
devmem 0x80110000 32 129   # 0x81 = ap_start | auto_restart

echo "Start TPG (auto-restart)"
devmem 0x800A0000 32 129   # 0x81 = ap_start | auto_restart

echo "Done."
```

运行脚本没有报错

```
root@petalinux:~# modetest -M xlnx -s 38@36:3840x2160-60@BG24&
root@petalinux:~# cat /sys/kernel/debug/dri/0/state | grep paddr
                                paddr=0x0000000037800000
root@petalinux:~# sh tpg_to_hdmi.sh 
Stop TPG
Stop FBWR
Configure TPG
Configure AXIS switch
Configure FBWR
Start FBWR
Start TPG (auto-restart)
Done.
```



但是没有显示tpg的图案,只是保持了modetest的图案

```shell
root@petalinux:~# devmem 0x80110000
0x0000000E
root@petalinux:~# devmem 0x80110000
0x00000004
root@petalinux:~# devmem 0x80110000
0x00000004
root@petalinux:~# devmem 0x800A0000                                                                                                                                                                                
0x0000008B
root@petalinux:~# devmem 0x800A0000
0x00000081
root@petalinux:~# devmem 0x800A0000
0x00000081
root@petalinux:~# devmem 0x800A0000
0x00000081
```

没有 timing，`FBWR` 永远只会写一帧. `v_framebuf_wr` 必须在 **video-timed 模式** 下工作.

`VTC` 提供 **连续、稳定的 timing**

***

## 添加`vtc`

### 路径1

参考`base_trd`, 也取消`axis_switch`

```
[VTC] -> [TPG] -> [v_framebuf_wr] -> PS DDR 
[CSI2-RX] -> [v_framebuf_wr] -> PS DDR    
                                                         
PS DDR -> [v_mix] -> HDMI
PS DDR -/

```

| git commit | xsa md5                          |
| ---------- | -------------------------------- |
| 24f27201   | d60a088d909a57e19c45656596ba5eb3 |

[system.pdf](24f27201/system.pdf)

[tpg_input.pdf](24f27201/tpg_input.pdf)

[hdmi_output.pdf](24f27201/hdmi_output.pdf)

[mipi_csi2_rx.pdf](24f27201/mipi_csi2_rx.pdf)

![Snipaste_2026-01-23_11-14-28](24f27201/Snipaste_2026-01-23_11-14-28.png)

![Snipaste_2026-01-23_11-15-33.png](24f27201/Snipaste_2026-01-23_11-15-33.png)

![Snipaste_2026-01-23_11-16-59.png](24f27201/Snipaste_2026-01-23_11-16-59.png)

![Snipaste_2026-01-23_11-29-26.png](24f27201/Snipaste_2026-01-23_11-29-26.png)

导入xsa, 编译测试

导入xsa硬件, 修改`project-spec/configs/config`, 修改`device-tree`排错

之后`fsbl-firmware`编译报错, 这样处理可以通过

```
petalinux-build -c fsbl-firmware -x do_clean
petalinux-build
```



### 路径2

修改路径如下, 这是参考了`vcu_trd`的

```bash
[VTC] -> [TPG] -> [v_framebuf_wr] -> PS DDR 
[CSI2-RX] -> [v_framebuf_wr] -> PS DDR    
                                                         
PS DDR -> [v_framebuf_rd] -> [v_mix] -> HDMI
                     PS DDR -/

```

| git commit | xsa md5hash                      |
| ---------- | -------------------------------- |
| 09b575e7   | eeff5e5cc68944db8a11f3e11f3d8a92 |

[system.pdf](09b575e7/system.pdf)

[tpg_input.pdf](09b575e7/tpg_input.pdf)

[hdmi_output.pdf](09b575e7/hdmi_output.pdf)	除了这里变化了其他没变

[mipi_csi2_rx.pdf](09b575e7/mipi_csi2_rx.pdf)

![Snipaste_2026-01-23_11-21-15.png](09b575e7/Snipaste_2026-01-23_11-21-15.png)

![Snipaste_2026-01-23_11-30-29.png](09b575e7/Snipaste_2026-01-23_11-30-29.png)



导入xsa硬件, 修改`project-spec/configs/config`, 修改`device-tree`排错

之后`fsbl-firmware`编译报错, 这样处理可以通过

```
petalinux-build -c fsbl-firmware -x do_clean
petalinux-build
```



查看

```
modetest -M xlnx -p
modetest -M xlnx
```

```bash
root@petalinux:~# modetest -M xlnx
Encoders:
id      crtc    type    possible crtcs  possible clones
40      0       TMDS    0x00000001      0x00000001

Connectors:
id      encoder status          name            size (mm)       modes   encoders
41      0       connected       HDMI-A-1        480x270         28      40
  modes:
        index name refresh (Hz) hdisp hss hse htot vdisp vss vse vtot
  #0 3840x2160 59.85 3840 3888 3920 4010 2160 2163 2168 2222 533250 flags: phsync, pvsync; type: preferred, driver
  #1 3840x2160 60.00 3840 4016 4104 4400 2160 2168 2178 2250 594000 flags: phsync, pvsync; type: driver
  #2 3840x2160 59.94 3840 4016 4104 4400 2160 2168 2178 2250 593407 flags: phsync, pvsync; type: driver
  #3 3840x2160 30.00 3840 4016 4104 4400 2160 2168 2178 2250 297000 flags: phsync, pvsync; type: driver
  #4 3840x2160 29.97 3840 4016 4104 4400 2160 2168 2178 2250 296703 flags: phsync, pvsync; type: driver
  #5 3840x2160 29.98 3840 3888 3920 4000 2160 2163 2168 2191 262750 flags: phsync, nvsync; type: driver
  #6 2560x1440 59.95 2560 2608 2640 2720 1440 1443 1448 1481 241500 flags: phsync, nvsync; type: driver
  #7 1920x1080 60.00 1920 2008 2052 2200 1080 1084 1089 1125 148500 flags: phsync, nvsync; type: driver
  #8 1920x1080 60.00 1920 2008 2052 2200 1080 1084 1089 1125 148500 flags: phsync, pvsync; type: driver
  #9 1920x1080 59.94 1920 2008 2052 2200 1080 1084 1089 1125 148352 flags: phsync, pvsync; type: driver
  #10 1920x1080 50.00 1920 2448 2492 2640 1080 1084 1089 1125 148500 flags: phsync, pvsync; type: driver
  #11 1680x1050 59.88 1680 1728 1760 1840 1050 1053 1059 1080 119000 flags: phsync, nvsync; type: driver
  #12 1600x900 60.00 1600 1624 1704 1800 900 901 904 1000 108000 flags: phsync, pvsync; type: driver
  #13 1280x1024 60.02 1280 1328 1440 1688 1024 1025 1028 1066 108000 flags: phsync, pvsync; type: driver
  #14 1440x900 59.90 1440 1488 1520 1600 900 903 909 926 88750 flags: phsync, nvsync; type: driver
  #15 1280x960 60.00 1280 1376 1488 1800 960 961 964 1000 108000 flags: phsync, pvsync; type: driver
  #16 1366x768 59.79 1366 1436 1579 1792 768 771 774 798 85500 flags: phsync, pvsync; type: driver
  #17 1280x800 59.91 1280 1328 1360 1440 800 803 809 823 71000 flags: phsync, nvsync; type: driver
  #18 1280x720 60.00 1280 1390 1430 1650 720 725 730 750 74250 flags: phsync, pvsync; type: driver
  #19 1280x720 59.94 1280 1390 1430 1650 720 725 730 750 74176 flags: phsync, pvsync; type: driver
  #20 1280x720 50.00 1280 1720 1760 1980 720 725 730 750 74250 flags: phsync, pvsync; type: driver
  #21 1024x768 60.00 1024 1048 1184 1344 768 771 777 806 65000 flags: nhsync, nvsync; type: driver
  #22 800x600 60.32 800 840 968 1056 600 601 605 628 40000 flags: phsync, pvsync; type: driver
  #23 720x576 50.00 720 732 796 864 576 581 586 625 27000 flags: nhsync, nvsync; type: driver
  #24 720x480 60.00 720 736 798 858 480 489 495 525 27027 flags: nhsync, nvsync; type: driver
  #25 720x480 59.94 720 736 798 858 480 489 495 525 27000 flags: nhsync, nvsync; type: driver
  #26 640x480 60.00 640 656 752 800 480 490 492 525 25200 flags: nhsync, nvsync; type: driver
  #27 640x480 59.94 640 656 752 800 480 490 492 525 25175 flags: nhsync, nvsync; type: driver
  props:
        1 EDID:
                flags: immutable blob
                blobs:

                value:
                        00ffffffffffff00212b010031303030
                        1f18010380301b782aa220a656489a24
                        12505421080081c0814081809500b300
                        a9c0810001014dd000aaf0703e803020
                        3500dd0c1100001e023a801871382d40
                        582c4500dd0c1100001a000000fc004d
                        585830380a20202020202020000000ff
                        004150574232484130323533303001d3
                        020328b14b9004031f13011202115f61
                        6b030c001000883c2000200167d85dc4
                        01788003e30f0006a36600a0f0701f80
                        302035000f282100001a662156aa5100
                        1e30468f33000f282100001e565e00a0
                        a0a02950302035000f282100001a0000
                        00000000000000000000000000000000
                        000000000000000000000000000000fe
        2 DPMS:
                flags: enum
                enums: On=0 Standby=1 Suspend=2 Off=3
                value: 3
        5 link-status:
                flags: enum
                enums: Good=0 Bad=1
                value: 0
        6 non-desktop:
                flags: immutable range
                values: 0 1
                value: 0
        4 TILE:
                flags: immutable blob
                blobs:

                value:
        8 GEN_HDR_OUTPUT_METADATA:
                flags: blob
                blobs:

                value:
        42 colorspace:
                flags: range
                values: 0 12
                value: 0
        43 ycbcr_enc:
                flags: range
                values: 0 8
                value: 0
        44 xfer_func:
                flags: range
                values: 0 7
                value: 0
        45 quantization:
                flags: range
                values: 0 2
                value: 0
        46 height_out:
                flags: range
                values: 480 2160
                value: 0
        47 width_out:
                flags: range
                values: 640 4096
                value: 0
        48 in_fmt:
                flags: range
                values: 4106 8448
                value: 0
        49 out_fmt:
                flags: range
                values: 4106 8448
                value: 0
        50 aspect_ratio:
                flags: range
                values: 0 3
                value: 0

CRTCs:
id      fb      pos     size
39      0       (0,0)   (0x0)
  #0  nan 0 0 0 0 0 0 0 0 0 flags: ; type: 
  props:
        25 VRR_ENABLED:
                flags: range
                values: 0 1
                value: 0

Planes:
id      crtc    fb      CRTC x,y        x,y     gamma size      possible crtcs
34      0       0       0,0             0,0     0               0x00000001
  formats: NV12
  props:
        9 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 0
35      0       0       0,0             0,0     0               0x00000001
  formats: YUYV
  props:
        9 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 0
36      0       0       0,0             0,0     0               0x00000001
  formats: UYVY
  props:
        9 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 0
37      0       0       0,0             0,0     0               0x00000001
  formats: AB24
  props:
        9 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 0
        33 alpha:
                flags: range
                values: 0 256
                value: 256
38      0       0       0,0             0,0     0               0x00000001
  formats: BG24
  props:
        9 type:
                flags: immutable enum
                enums: Overlay=0 Primary=1 Cursor=2
                value: 1

Frame buffers:
id      size    pitch


```



# 迁移`vcu_trd`或`base_trd`到目标板子或开发板

先不考虑从自定义工程出发修改, 而是从`trd`进行简化

`zcu102`的`base_trd`比较老, 没有2022.2版本的, 目前只能通过虚拟机来编译.  结构上最接近目标板设计, 除了vcu.

![img](file:///home/andy/workdir/trd/base_trd_block_diagram.jpg)

`zcu106`的`vcu_trd`有2022.2版本的, 至少需要在开发板先跑起来再简化.

![img](file:///home/andy/workdir/trd/vcu_trd_block_diagram.png)





