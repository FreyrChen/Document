### 1. 编译前准备工作
* 准备一个Ubuntu16.04 的虚拟机，并安装下面的软件。
```
sudo apt install uuid uuid-dev zlib1g-dev liblz-dev  liblzo2-2 liblzo2-dev   lzop   -y
sudo apt-get install git-core curl  u-boot-tools  mtd-utils  android-tools-fsutils device-tree-compiler     gdisk    m4  libxml2-utils   -y
sudo apt-get install libc6:i386 libncurses5:i386 libstdc++6:i386 -y
sudo apt-get install openjdk-8-jdk -y
```

### 2. 编译脚本
* 下面的脚本配置一下 
```
#!/bin/bash

help_function() {
    echo "$0  update       : create update.img"
    echo "$0  uboot        : compiler uboot"
    echo "$0  kernel       : compiler kernel"
    echo "$0  android      : compiler uboot"

    exit 1
}

if [ ! $# -eq 1 ]; then
    help_function
fi

cd ${ANDROID_PATH}
source build/envsetup.sh
lunch rk3399_all-userdebug

TARGET_PRODUCT=`get_build_var TARGET_PRODUCT`
DEVICE=`get_build_var TARGET_PRODUCT`
BUILD_VARIANT=`get_build_var TARGET_BUILD_VARIANT`
PLATFORM_VERSION=`get_build_var PLATFORM_VERSION`

PACK_TOOL_DIR=RKTools/linux/Linux_Pack_Firmware
IMAGE_PATH=rockdev/Image-$TARGET_PRODUCT

#set jdk version
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
cd -

export TARGET_PATH=${ANDROID_PATH}/out/target/product/rk3399_all/

uboot_compiler() {
        cd ${ANDROID_UBOOT_PATH}
    make distclean
        make rk3399_defconfig
        make ARCHV=aarch64 -j20
        cd -
}

kernel_compiler() {
    cd $ANDROID_KERNEL_PATH
    make distclean
    #make ARCH=arm64 rockchip_defconfig
    #make ARCH=arm64 rk3399-sapphire-excavator-edp.img -j20
    make ARCH=arm64 cmi_at157_android_defconfig
    make ARCH=arm64 rk3399_cmi_ar157.img -j20

    cd -
}

android_compiler() {
    cd $ANDROID_PATH
    source build/envsetup.sh
    lunch rk3399_all-userdebug
    make installclean
    make -j20

    if [ $? -eq 0 ]; then
        echo "Make image ok!"
    else
        echo "Make image failed!"
        exit 1
    fi
    cd -
}

create_update_image() {
    cd $ANDROID_PATH
    ./mkimage.sh

    #cp ${TARGET_PATH}/*.img   ${IMAGE_PATH}/  -rf

    #cp u-boot/uboot.img $IMAGE_PATH/
    #cp u-boot/RK322XHMiniLoaderAll* $IMAGE_PATH/
    #cp u-boot/trust.img $IMAGE_PATH/

    #cp kernel/resource.img $IMAGE_PATH/
    #cp kernel/kernel.img $IMAGE_PATH/

    echo "copy manifest.xml"
    cp manifest.xml $IMAGE_PATH/manifest_${DATE}.xml

    echo "generate update.img"
    unzip -o  $PACK_TOOL_DIR/Linux_rockdev_2015-06-17_for_RK3399.zip -d ${IMAGE_PATH}/../update_gen
    cp ${IMAGE_PATH}/parameter.txt ${IMAGE_PATH}/../update_gen/rockdev/parameter.txt -rf
    cp ${IMAGE_PATH}/parameter.txt ${IMAGE_PATH}/../update_gen/rockdev/parameter -rf
    cp ${IMAGE_PATH}/trust.img ${IMAGE_PATH}/../update_gen/rockdev/trust.img -rf
    cp ${IMAGE_PATH}/uboot.img ${IMAGE_PATH}/../update_gen/rockdev/uboot.img -rf
    cp ${IMAGE_PATH}/MiniLoaderAll.bin ${IMAGE_PATH}/../update_gen/rockdev/RK3399MiniLoaderAll_V1.05.bin -rf
    cp ${IMAGE_PATH}/*      ${IMAGE_PATH}/../update_gen/rockdev/Image/ -rf
    cd ${IMAGE_PATH}/../update_gen/rockdev
    chmod +x ./mkupdate.sh
    chmod +x ./afptool
    chmod +x ./rkImageMaker
    chmod +x ./unpack.sh
    ./mkupdate.sh

    if [ $? -eq 0 ]; then
        echo "Make update image ok!"
    else
        echo "Make update image failed!"
        exit 1
    fi
    cd -
    cp ${IMAGE_PATH}/../update_gen/rockdev/update.img ~ -rf
    #cp ${IMAGE_PATH}/../update_gen/rockdev/update.img ${IMAGE_PATH}/update.img
    rm -rf ${IMAGE_PATH}/../update_gen
    sync
    ls ${IMAGE_PATH}/*.img -lh
    ls ~/update.img -lh

    CUSTOM_NAME="aplex"
    #CUSTOM_NAME="test"

    BOARD_VERSION="CMIAR157R010"

    SYSTEM_VERSION="A7.1V001"

    if [ ${CUSTOM_NAME} == "aplex" ]; then
        CUSTOM=""
    elif [ ${CUSTOM_NAME} == "test" ]; then
        echo "test"
    else
        echo "else"
    fi

    STATUS="Debug"
    YEAR="20"
    MD=`date "+%m%d"`

    IMAGE_NAME=${BOARD_VERSION}"_"${SYSTEM_VERSION}"_"${CUSTOM}${YEAR}${MD}"_"${STATUS}".img"

    echo ${IMAGE_NAME}

}

clean_all() {
    cd $ANDROID_PATH
    cd -
}

echo $1

if [ $1 = "all" ]; then
    uboot_compiler
    kernel_compiler
    android_compiler
    create_update_image
elif [ $1 = "uboot" ]; then
    uboot_compiler
elif [ $1 = "kernel" ]; then
    kernel_compiler
elif [ $1 = "android" ]; then
    android_compiler
elif [ $1 = "update" ]; then
    create_update_image
elif [ $1 = "clean" ]; then
    clean_all
else
    help_function
fi

```