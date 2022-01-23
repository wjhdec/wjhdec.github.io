# Jetty安装脚本


　 　从头部署jetty比较麻烦，这里写了一个安装jetty的脚本，可以比较快速的部署jetty。

<!-- more -->

## 主要功能

* 创建用户及组
* 解压jetty
* 配置程序与工作目录分离
* 自动复制配置文件及需要信息到工作目录
* 添加服务到自启动（centos系统，ubuntu等系统需自己改，很简单）

## 自己配置

* 端口等配置须自己设置
* 防火墙各个系统的不一样，须自己开端口

## 脚本主体

``` bash
#!/bin/bash
echo "安装jetty"

##################################################
# 参数设置
##################################################

# 安装目录
installPath=/opt/apps/jetty
# 工作目录
JETTY_BASE=/opt/jettyWebs
USER=jetty
GROUP=jetty
# 压缩包路径，tar.gz包
JettyPath=jetty-distribution-9.4.6.v20170531.tar.gz

##################################################
# 公共方法
##################################################

function exitInfo()
{
    echo "退出 ： $1"
    exit 0
}

function createUser()
{
    egrep "^$GROUP" /etc/group >& /dev/null  
    if [ $? -ne 0 ]  
    then
        echo "添加用户组$GROUP"
        groupadd $GROUP  
    fi  
    
    egrep "^$USER" /etc/passwd >& /dev/null  
    if [ $? -ne 0 ]  
    then
        echo "添加用户$USER"
        useradd -g $GROUP $USER  
    fi  
}

function deleteFolder()
{
    if [ -d "$1" ]; then 
        rm -rf "$1"
        echo "删除文件夹: $1"
    fi
}

function createFolder()
{
    if [ ! -d "$1" ]; then 
        mkdir -p "$1"
        echo "创建文件夹: $1"
    fi
}

##################################################
# jetty相关方法
##################################################

function createJettyDic()
{
    createFolder "$installPath"
    createFolder "$JETTY_BASE"
}

function unzipJetty()
{
    jettyFile=${JettyPath##*/}
    jettyFileName=${jettyFile/%.tar.gz/}

    tar -xzvf "$JettyPath" -C "$installPath" 1>/dev/null 2>&1
    echo "解压到 $installPath"
    lnsPath=$installPath/current
    rm -f $lnsPath
    ln -s "$installPath/$jettyFileName" "$lnsPath"

    if [ ! -f "$JETTY_BASE/start.ini" ]; then  
        cp "$lnsPath"/start.ini "$JETTY_BASE"
    fi
    createFolder "$JETTY_BASE"/temp
    createFolder "$JETTY_BASE"/work

    createFolder /var/run/jetty

    cp "$lnsPath"/demo-base/* "$JETTY_BASE"

    chown -R "$USER":"$GROUP" "$installPath"
    chown -R "$USER":"$GROUP" "$JETTY_BASE"
    chown -R "$USER":"$GROUP" /var/run/jetty

    usermod -d $JETTY_BASE $USER

    cp -f "$lnsPath"/bin/jetty.sh /etc/init.d/jetty

    chmod 700 /etc/init.d/jetty
    echo -e "JETTY_HOME=$lnsPath\nJETTY_BASE=$JETTY_BASE\nTMPDIR=$JETTY_BASE/temp\nJETTY_USER=$USER" > /etc/default/jetty

    # centos自启，其他系统改这里
    chkconfig jetty on
}

if [ "$(whoami)" != "root" ]
then
    exitInfo "请用root用户或用 sudo 安装"
fi

#添加用户
createUser
# 添加文件夹
createJettyDic
# 解压jetty
unzipJetty
```
