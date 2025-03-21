---
title : os 系统
---


# 自启动 使用 systemmd 服务
- 在petalinux或yocto构建的系统中 
.service 文件定义服务的启动、停止、依赖关系、运行环境等
.bb (BitBake) 配方文件，用于定义如何获取、编译、安装一个软件包，并将其集成到镜像中

- 自启动的流程
BitBake 构建系统 → 根据 .bb 文件安装服务 → 镜像打包 → 系统启动 → systemd 加载服务 → 按依赖顺序启动服务
- 详细流程
1. BitBake 构建阶段
通过 .bb 文件中的 do_install 将可执行程序和服务文件安装到临时目录（${D}）。
FILES 变量声明哪些文件需要被打包到最终的根文件系统镜像（如 hawaii_algorithm 和 hawaii_algorithm.service）。

2. 镜像生成阶段
所有通过 FILES 声明的文件被包含在镜像中，例如：
/usr/bin/hawaii_algorithm
/lib/systemd/system/hawaii_algorithm.service

3. 系统启动阶段
Linux 内核启动后，由 systemd 作为初始化系统（PID=1）接管。
systemd 读取 /lib/systemd/system/ 中的服务文件，并根据 WantedBy 配置将服务链接到对应的目标（如 multi-user.target）。
根据依赖关系（After= 和 Requires=）按顺序启动服务。

4. 服务激活过程
当系统进入 multi-user.target 时，systemd 启动所有与该目标关联的服务。
服务按依赖顺序启动：例如 network.target 先于 hawaii_algorithm.service 启动。

### create app
1. petalinux-create -t apps --template install --name service_name --enable 创建一个app服务
### file path
2. 到项目的 project-spec/meta-user/recipes-apps/service_name/files 目录下 创建想要添加的自启动 .service文件(自动生成的 service_name 文件可以不用管)
### 服务编写
3. 编写 .service 文件，以autostart为例
需要启动两个服务 一个是算法解析模块 一个是应用模块
这里将我的可执行程序拷贝到了files目录下 以便.bb文件 生成配方把文件放到指定的目录

> 算法模块
```service
[Unit]
Description=Hawaii Algorithm Service  
After=network-online.target  # 等待网络完全就绪（需 systemd-networkd-wait-online）
After=serial-getty@ttyPS0.service  # 等待串口 Getty 服务启动
Requires=network-online.target      # 强制依赖网络就绪
Requires=serial-getty@ttyPS0.service  # 强制依赖串口服务

# After=multi-user.target            # 在系统进入多用户模式后启动

[Service]
Type=simple
ExecStart=/home/petalinux/hawaii_algorithm  # 可执行程序的install目录 取决于你的.bb文件
Restart=on-failure
User=root
StandardOutput=journal+console       # 输出到日志和控制台
StandardError=journal+console

# 如果目录不存在，启动前自动创建
ExecStartPre=/bin/mkdir -p /home/petalinux
ExecStartPre=/bin/chmod 755 /home/petalinux

[Install]
WantedBy=multi-user.target
```

> 应用模块
```service
[Unit]
Description=Hawaii autostart service
After=hawaii_algorithm.service  # 明确依赖 hawaii_algorithm
Requires=hawaii_algorithm.service  # 强制依赖

[Service]
Type=simple
# ExecStart=/etc/init.d/autostart start  # 直接调用原有脚本
# 或直接执行命令（推荐）：
ExecStart=/home/petalinux/hawaii 
Restart=on-failure
User=root
StandardOutput=journal+console       # 输出到日志和控制台
StandardError=journal+console


# 如果目录不存在，启动前自动创建
ExecStartPre=/bin/mkdir -p /home/petalinux
ExecStartPre=/bin/chmod 755 /home/petalinux

[Install]
WantedBy=multi-user.target  # 设置启动目标
```
### 配方编写
4. 编写.bb(BitBake) 文件
```bb
#
# This file is the autostart recipe. 
#

SUMMARY = "Simple autostart application" 
# SECTION = "PETALINUX/apps" 
LICENSE = "MIT" 
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302" 


SRC_URI = "file://hawaii_algorithm \
           file://hawaii \
           file://hawaii_algorithm.service \
           file://hawaii.service"

# 继承 systemd 类以支持服务管理
inherit systemd

# 启用 systemd 服务
SYSTEMD_SERVICE:${PN} = "hawaii_algorithm.service hawaii.service"

S = "${WORKDIR}"

do_install() {
    # 创建目录
    install -d -m 0755 ${D}/home/petalinux

    # 安装可执行程序到 /home/petalinux
    install -m 0755 ${S}/hawaii_algorithm ${D}/home/petalinux/
    install -m 0755 ${S}/hawaii ${D}/home/petalinux/

    # 安装 systemd 服务文件到 /lib/systemd/system
    install -d ${D}${systemd_system_unitdir}
    install -m 0644 ${S}/hawaii_algorithm.service ${D}${systemd_system_unitdir}/
    install -m 0644 ${S}/hawaii.service ${D}${systemd_system_unitdir}/

}

# 定义安装后的文件路径（关键！）
FILES:${PN} += " \
    ${systemd_system_unitdir}/hawaii.service \
    ${systemd_system_unitdir}/hawaii_algorithm.service \
    /home/petalinux/hawaii \
    /home/petalinux/hawaii_algorithm \
"
```


# 




