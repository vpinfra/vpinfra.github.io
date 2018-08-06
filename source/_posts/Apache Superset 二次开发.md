# Apache Superset 二次开发

标签（空格分隔）： 前端

---

首先介绍一下**superset**，Superset 是 Airbnb 开源的一个旨在视觉，直观和交互式的数据探索平台（曾用名 Panoramix、Caravel，现已进入 Apache 孵化器），给上github链接:[点这里](https://github.com/apache/incubator-superset)

然后说一下为什么要对superset进行二次开发，是因为随着报表越来越多，导致dashboard里有N多的分类,

然而一个dashboard只能放某一类的报表，所以dashboard的数量就越积越大，支持dashboard group就迫在眉睫。

某一天我在逛github的时候，偶然发现一个大神在某一分支上有打算开发这个功能，给上链接:[点这里](https://github.com/apache/incubator-superset/tree/dashboard-builder),就打算直接把代码拉下来看看能不能用

拉下来的代码如下：
![](https://raw.githubusercontent.com/YangDaWang/image/master/image/20180711190932.png)

## 准备工作 ##
1. Linux   Ubuntu 
2. python环境：python3.6.5，这里我建议用`anaconda`，很方便又强大
3. pip
```shell
pip install --upgrade setuptools pip
```
##开始##
```shell
    cd /opt/new_superset/incubator-superset-master/superset/static
    mv assets assets_bak
    ln -s ../assets assets
    cd /opt/new_superset/incubator-superset-master/
    python setup.py develop
    pip freeze | grep superset
```    
![](https://raw.githubusercontent.com/YangDaWang/image/master/image/20180711193201.png)
此时会显示你正在开发的superset版本号
这时把你的配置文件设置到环境变量里，官网所说的PYTHONPATH是没有用的，需要设置SUPERSET_CONFIG_PATH

> export SUPERSET_CONFIG_PATH=/opt/new_superset/incubator-superset-master/superset/superset_config.py

```shell
    export SUPERSET_CONFIG_PATH=/opt/new_superset/incubator-superset-master/superset/superset_config.py
    source /etc/profile
```

这时superset启动时就能读到你写的配置文件了

接下来是根据配置文件初始化数据库

```shell
superset db upgrade --升级数据
superset init --初始化数据库
```

**注意，这里如果没有设置好配置文件的话，默认读的初始化的数据库是本地的SQLite，之前如果有使用过superset的话将根据配置文件读相应的数据库，然后会增加或修改一些字段，这个操作不可逆，就是说一旦升级，那么之前的旧版本的superset将在查询的时候报错**

```shell
superset runserver -d --启动superset
```

此时，可以通过http://127.0.0.1:8088/去访问superset，但是输入密码登陆之后会发现CSS样式是不加载的，所以我们还需要启动node服务用来渲染前端

## 准备工作 ##

安装nodejs和npm

```shell
wget https://npm.taobao.org/mirrors/node/v8.0.0/node-v8.0.0-linux-x64.tar.xz    --下载nodejs
tar -xvf  node-v8.0.0-linux-x64.tar.xz    --解压
cd  node-v8.0.0-linux-x64/bin && ls    --进入解压目录下的 bin 目录，执行 ls 命令
./node -v  --测试
```    
    
![](https://raw.githubusercontent.com/YangDaWang/image/master/image/20180711194917.png)

安装成功
现在 node 和 npm 还不能全局使用，做个链接
```shell
ln -s /opt/new_superset/incubator-superset-master/superset/assets/node-v8.0.0-linux-x64/bin/node /usr/bin/node
ln -s /opt/new_superset/incubator-superset-master/superset/assets/node-v8.0.0-linux-x64/bin/npm /usr/bin/npm
```

    
    
OK，现在node和npm可以全局使用

```shell    
cd /opt/new_superset/incubator-superset-master/superset/assets
npm install -g yarn --安装依赖 --安装yarn代替npm更快更稳定
yarn --运行yarn获取所有依赖项
```

最后,启动nodejs
```shell
npm run dev
```




