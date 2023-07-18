
# 〇、基础介绍
## 1.硬件配置
恩兔N2-NS1采用海思hi3798mv200芯片，四核A53，主频1.6G，单将CPU性能，比RTD1296还强一些，内存采用ddr4 2166，1G，存储为mmc 8G。hi3798支持原生sata、usb3.0和pcie2.0.但其PHY是复用的，所以恩兔引出了sata，也就不支持usb3.0了，同时该soc支持内置2个千兆mac，一个百兆phy，n2用的是千兆mac连接外置千兆PHY。
## 2.关于原厂系统
n2原厂系统经验证已经不可用，无法绑定，所以不刷机就不能用了，原厂采用的32位linux系统，所以性能也受到一定限制。
## 3.关于已有的小钢炮
壳大做了小钢炮固件，按说比较易用，之所以重新制作，一是小钢炮没有提供gpio操作，开机后U盘不供电（硬盘未测试），二是对于我这种有洁癖的希望做一个64位系统，自主安装软件。
## 4.关于刷机包文件（以debian为例）
### n2ns1_debian.xml
分区表文件，共有6个分区
### fastboot.bin 
uboot文件，由sdk编译而成，并替换了原厂的reg文件，reg文件是外设寄存器配置，主要是引脚复用等，同时该fastboot主体是32位，sdk设置为64位，但uboot只有32位，主要是通过在编译内核时附加atf，在引导阶段将cpu切换到64位模式。
### Bootargs
对用的uboot的env文件，就是引导时的命令和内核参数
### stock_kernel
原厂内核文件，为原厂固件提取的32位内核
### stock_squash
原厂rootfs，并经过修改，因为原厂rootfs串口不正常，ttl看不到登录界面，原因是gettty使用了错误的串口号，已经修正，可以ttl登录，也可以直接telnet，root用户没有密码，作为恢复和装系统使用的临时系统。同时停止原厂云应用运行。
### hi_kernel.bin
内核文件，由SDK编译而成，是64位内核，集成了dtb，同时整个内核和dtb作为atf的载荷，在boom命令执行后，由atf切换到64位模式并引导内核执行。所有gpio和led、U盘和硬盘供电的引脚已经通过逆向得到并写入dtb，开机后U盘和硬盘自动供电，电源按钮可用，led可以通过sysfs操作，实现开关led和闪烁等功能。
### stretch.tar.bz2 
debian的rootfs，由debian9生成，为什么选择debian9，而不是10和11，因为sdk内核版本为4.4.35，如果换用10或11，libc版本较高，会封装没在4.4内核的系统调用，造成启动阶段systemd提示一些syscall未找到，我有点洁癖，见不得提示错误，经过验证debian10可以正常运行，会提示syscall271未找到，debian11提示就比较多了。同时该rootfs已经继承了内核编译生成的module和header，可以在线编译新的软件和模块。


# 一、刷机准备
1.硬件连接
准备ttl转USB，TTL排线，TTL插针和网线，插针插上即可无需焊接

Ttl连接顺序，连接靠近指示灯一侧额四个孔，最右侧方孔为VCC，不要连接，
从左至右（靠近方孔为右侧）依次为GND、RX、TX，特别注意RX接TTL转接板的TX，TX接TTL转接板的RX，如下图所示：

![image](https://github.com/xiayang0521/n2ns-1/assets/23094327/491d87cd-ee77-45fa-909a-8c104eab384a)
![48f1ef86b584b812bee261baac28d74](https://github.com/xiayang0521/n2ns-1/assets/23094327/7c91ce97-c26d-4c10-be32-09853a3755f2)
![ef5cdde1a352c2f1659bc257feebfcd](https://github.com/xiayang0521/n2ns-1/assets/23094327/7018ae61-6042-43c2-84b9-31539acf14ad)


2.打开hitool，配置芯片为hi3798mv200，运行hiburn，链接TTL之后点击刷新，一般自动识别服务器IP（电脑IP），
查看串口号，修改为TTL的COM号，比如为COM7；N2的板端地址IP和电脑IP必须同一局域网，比如开机插网线连路由查看
N2的IP192.168.1.10，电脑IP为192.168.1.18，
即板端地址为192.168.1.10，服务器IP为192.168.1.18；点击浏览，选择对应的xml分区文件

![a284a67be6ab617b4e6c5c4b58ca288](https://github.com/xiayang0521/n2ns-1/assets/23094327/6f025e3d-112c-466c-955f-000039bc1a65)



3.烧录
选择emmc烧录，点击浏览，选择分区表XML文件，并勾选除了rootfs之外的分区


# 二、刷机（debian/centos）rootfs准备
## 1.说明
本刷机包没有直接刷入rootfs，因为太大，所以刷入一个原厂系统，直接解压debian系统。
## 2.保持主板断电
点击烧写，根据下侧窗口提示上电

## 3.等待烧写完毕
即可断电，时间因该2分钟左右，因为，只有原厂系统
## 4.将以下文件
stretch.tar.bz2，CentOS-7-aarch64-7.5.1804-k4.4.35-hi3798mv2x.img.xz和bootargs2 文件放入U盘根目录，机器连接网线和U盘开机
## 5.在路由器找到设备ip
用telnet连接用户名root（注意不是ssh），密码为空

![1689474738714](https://github.com/xiayang0521/n2ns-1/assets/23094327/b87b9f54-21c3-4b59-b152-a6c740ec6f73)


## 6.开启U盘供电
````
echo 33 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio33/direction
echo 1 > /sys/class/gpio/gpio33/value
此时再用blkid就能看到U盘了
````
##  7.挂载U盘和emmc
````
mount /dev/sda1 /mnt/usb1
````

# 三、可刷系统（有的未测）
##  1. Debian10系统

改变启动参数，下次重启从debian启动
dd if=/mnt/usb1/bootargs2 of=/dev/mmcblk0p2

安装debian到emmc
````
mkfs.ext4 /dev/mmcblk0p6
mkdir /tmp/mmc
mount /dev/mmcblk0p6 /tmp/mmc
tar xvjpf /mnt/usb1/stretch.tar.bz2 -C /tmp/mmc

重启
reboot
````
使用
````
ssh连接，重启后即可通过ssh连接，用户名root，密码shaoxi
led操作
可以看到8个gpio的led全部注册正常，
打开
echo 255 > /sys/class/leds/green:fn/brightness
关闭
echo 0 > /sys/class/leds/green:fn/brightness
触发，如闪烁、管理mmc读写等

可以通过cat命令可以看到led支持的触发方式，通过echo回写相应的字符串可以实现led的自动触发，如heartbeat代表闪烁，mmc0代表mmc0的读写触发led等等
关于内置软件
已经安装了samba、aria、nginx、php的常用软件，直接搜索debian配置即可
关于docker，没有内置docker，但是内核编译已经启用了docker支持，主要考虑没有硬盘的情况下，4G的空间不够docker用，可自行一键安装docker
apt-get install curl
curl -fsSL [https://get.docker.com](https://links.jianshu.com/go?to=https%3A%2F%2Fget.docker.com) | bash -s docker --mirror Aliyun
安装完成后，最好先停止docker，然后将docker的数据目录链接到硬盘某个目录下，例如
安装硬盘后,通过parted分区后，挂载在/sata目录，然后把/var/lib/docker软链接至/sata/docker。
修改bootargs参数，已经安装了uboot-tools，并配置了bootargs分区信息，可以直接通过fw_printenv打印启动参数，通过fw_setenv设置启动参数，如图通过设置bootcmd可以改变启动debian或者恢复系统
````
````
username:         root         
password:         shaoxi         
````

参见 https://www.wdmomo.fun:81/doc/index.html?file=003_%E5%90%84%E7%B1%BB%E7%9F%BF%E6%B8%A3/013_%E6%81%A9%E5%85%94NS-1

##  2. CentOS-7-aarch64-7.5.1804-k4.4.35-hi3798mv2x.img.xz
安装CentOS到emmc
````
将CentOS-7-aarch64-7.5.1804-k4.4.35-hi3798mv2x.img.xz和bootargs2文件放入U盘根目录，机器连接网线和U盘开机
dd if=/mnt/usb1/bootargs2 of=/dev/mmcblk0p2
mkfs.ext4 /dev/mmcblk0p6
xzcat CentOS-7-aarch64-7.5.1804-k4.4.35-hi3798mv2x.img.xz | sudo dd of=/dev/mmcblk0p6 bs=4M conv=sync

````
````
username:         root               n1
password:         centos           phicomm
````
##  3. N2_alpine3.17.img.7z
未测试
##  4. 小钢炮系统entu_xgp_emmc_image.7z（http://rom.nanodm.net/    https://nanodm.net/）
未测试
##  5. TTL-hi3798mv200-202304-64-n2ns1.zip 海纳思 hinas 固件(https://dl.ecoo.top/)
按照官网教程即可
````
username:         root         
password:       ecoo1234
````
##  6. Synology-n2ns1.zip群晖体验版 50分钟重启一次（无安装教程，作者雪狐）
