
[Hi3559V200](http://www.yeiey.com/index.php/post/45.html)虽然CPU只是900M的Cortex-A7双核，但具有4K@30fps编解码能力，内置实时EIS稳像，且有单独的NNIE核（0.4TFlops)，能够支持简单AI网络的实时运行，而且两个核可以分别运行linux及实时liteos。价格也十分便宜，很适合低成本AI机器人等领域。

本文简单总结Hi3559V200 SDK的基本操作，主要内容如下:
  - **[基于docker编译镜像](#how-to-build)**
  - **[如何烧写镜像](#how-to-burn)**
  - **[如何调试](#how-to-debug)**
	  - [开启usb网络](#enable-usb-network)
	  - [挂载nfs](#mount-nfs)
	  - [liteos系统调试](#debug-liteos)

# how to build
- 海思官方手册要求使用ubuntu14.04及gcc 4.8.2，为防止环境不一致引入的问题，采用docker进行编译，下面使用的dockerfile已上传至[github](https://github.com/LyuOnLine/dockerfile/tree/master/Hisi/Hisi3559)。
- 使用方法:
  - 编译docker image:
    ```
    git clone https://github.com/LyuOnLine/dockerfile
    cd Hisi/Hisi3559
    docker build -t lvjf/hisi .
    ```
  - 通过docker_run.sh，编译：
    ```
    # 进入Hi3559V200 SDK目录
    cd Hi3559V200_MobileCam_SDK_V1.0.1.5
    docer_run.sh make all
    ```
  - 通过source指令，加载docker:
    - 在Hi3559V200 SDK根目录，生成set_env.sh，源码如下:
      ```
      #!/usr/bin/env bash

      unset -f mm 1>/dev/null 2>&1
      ttyname=$(tty | tr -d '\r' |sed -e 's/\//_/g')
      dockername="mm${ttyname}"
      mm() {
        docker rm -f ${dockername} 1>/dev/null 2>&1

        docker run -it -v /etc/passwd:/etc/passwd -v /etc/group:/etc/group -v /home/$(whoami)/hhd2:/home/$(whoami)/hhd2 -v /home/$(whoami):/home/$(whoami) -v $(pwd):$(pwd) -w $(pwd) \
            -u $(id -u):$(id -g) --name ${dockername} --rm lvjf/hisi \
            bash -c "cd $(pwd) && $*"
        # in case of hb container not exited normally
        docker rm -f ${dockername} 1>/dev/null 2>&1
      }
      ```
    - 运行方法:
      ```
      cd Hi3559V200_MobileCam_SDK_V1.0.1.5
      source ./set_env.sh
      mm make all
      ```

# how to burn
- SD卡烧写， 会更新uboot及全部镜像。
  - 把SD卡格式化为FAT32文件格式。
  - 把镜像拷贝到SD卡根目录:
    ```
    cp reference/out/hi3559v200_actioncam_demb_imx458/burn/spinor/* /mnt/sdcard/
    ```
  - 把SD卡插入Hi3559主板，并按键选择SD BOOT方式，系统会自动进入烧写状态。

# how to debug

### enable usb network
- 默认镜像，主板未加载usb network驱动，需要手动加载。
  ```
  cd /app/komod
  ./usb2net_load.sh
  ```
- 静态配置hi3559主板IP:
  ```
  ifconfig usb0 192.168.3.13 netmask 255.255.255.0
	route add default gw 192.168.3.1
  ```
- host端配置网络
  - host配置静态IP
    ```
    sudo ifconfig usb0 192.168.3.1 netmask 255.255.255.0 up
    ```
  - 设置桥接，使hi3559主板能够访问外网
    ```
    sudo iptables -t nat -A POSTROUTING -s 192.168.3.13/24 -o eth1 -j MASQUERADE
    ```


### mount nfs

- host端配置nfs server:
  ```
  apt-get install nfs-server
  # 配置指定export目录
  sudo vi /etc/exports
  /home/lvjianfeng/work/Hi3559V200_MobileCam_SDK_V1.0.1.5 192.168.3.0/24(rw,sync,no_subtree_check)
  ```
- kernel配置，开启nfs:
  - 修改hi3559v200_amp_defconfig，增加nfs相关驱动:
    ```
    -# CONFIG_NETWORK_FILESYSTEMS is not set
    +CONFIG_NETWORK_FILESYSTEMS=y
    +CONFIG_NFS_FS=m
    +CONFIG_NFS_V2=m
    +CONFIG_NFS_V3=m
    ```
  - 重新编译烧写镜像。
  - 加载driver:
    ```
    cd /app/komod/
    insmod sunrpc.ko
    insmod grace.ko
    insmod lockd.ko
    insmod nfs.ko
    insmod nfsv2.ko
    insmod nfsv3.ko
    ```
- mount host目录:
  ```
  mkdir -p /mnt/host
  mount -t nfs -o tcp,nolock 10.13.3.1:/home/lvjianfeng/work/Hi3559V200_MobileCam_SDK_V1.0.1.5 /mnt/host
  ```

### debug liteos

- 在linux侧，可以通过virttty登录到liteos侧
  ```
  # 登录liteos
  virt-tty a7
  # 显示可以命令
  help
  # 退出
  Ctrl+C
  ```
- 显示liteos侧mpp错误日志:
  ```
  cat /dev/logmpp
  ```
