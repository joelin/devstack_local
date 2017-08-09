# 企业内部网络部署devstack
由于安全原因，企业内部无法访问互联网,有时也由于互联网出口的有限，把多人协作需要频繁访问的互联网资源同步到企业内网。面对这样的环境，如何在内部网络部署最新的openstack版本？此文介绍如何实现。

## 前提条件
熟读 `devstack` 的安装手册，了解到 `devstack` 安装，除了 `openstack` 的所有源码模块外，还需要操作系统源，`pip` 源，镜像文件下载地址。
为了在企业内网安装 `openstack`，需要把这些前提条件都准备，让其在企业内网可用。

### 操作系统源
以　`CENTOS` 为例，需要准备　`CENTOS` 源、`epel` 源，一般使用 `rsync` 的方式进行同步。参考链接:    https://wiki.centos.org/zh/HowTos/CreatePublicMirrors?highlight=%28rsync%29

准备完成应该可以提供如下访问方式：　

https://mirrors.aliyun.com/centos/7.3.1611/

https://mirrors.aliyun.com/epel/7/

* 源配置
需要配置如下源
   
    * `centos base`源
    * `openstack`源(https://mirrors.aliyun.com/centos/7.3.1611/cloud/x86_64/openstack-{version}), 安装其它组件，比如 `openvswitch` 
    * `epel` 源 

### pip源

`pip` 源有两种方式，有资源准备全量源更好，没有的话使用按需下载的方式。

#### 全量源
`pypi` 源搭建可以选择多种方式，比如　`pypi-mirror`　有选择性的同步需要的模块，　`rsync`、`bandersnatch`　同步所有的模块。

准备完毕后，应该可以提供如下访问方式：　
https://mirrors.aliyun.com/pypi

#### 按需下载源
根据 `openstack` 的 `requirements` 项目，找到所要的 `pip` 模块，https://github.com/openstack/requirements

使用 `pip2pi` 工具下载这些模块，并使其成为 `pip` 源。参考: https://github.com/wolever/pip2pi

由于有些模块需要编译安装，所以请安装开发者工具包，最好全量的。这里补充特殊的案例

>sudo yum -y install nss-devel nspr-devel python-devel

*  补充

    安装过程中，发现还缺少的包,大概为测试相关，可以在 `requirements/blacklist.txt` 里找到，具体版本需要看各个模块的需要，以下为 `2017-08-08` 的 `master` 为基准的版本。
    ```
    os-testr>=0.8.2
    hacking==0.12.0
    tempest==16.1.0
    pep257==0.7.0
    flake8-docstrings==0.2.1.post1
    flake8-import-order==0.12
    pylint==1.4.5

    openstack-requirements # tempest 使用
    ```
### 镜像地址
下载 `devstack` 里需要使用的镜像文件，到内网地址，一般由用户自己选择需要使用的镜像，默认镜像为外网，无法自定义，所以把默认镜像 `disable` 掉，
源码里默认使用的镜像为：

```
CIRROS_VERSION=${CIRROS_VERSION:-"0.3.5"}
CIRROS_ARCH=${CIRROS_ARCH:-"x86_64"}

# Set default image based on ``VIRT_DRIVER`` and ``LIBVIRT_TYPE``, either of
# which may be set in ``local.conf``.  Also allow ``DEFAULT_IMAGE_NAME`` and
# ``IMAGE_URLS`` to be set in the `localrc` section of ``local.conf``.
DOWNLOAD_DEFAULT_IMAGES=$(trueorfalse True DOWNLOAD_DEFAULT_IMAGES)
if [[ "$DOWNLOAD_DEFAULT_IMAGES" == "True" ]]; then
    if [[ -n "$IMAGE_URLS" ]]; then
        IMAGE_URLS+=","
    fi
    case "$VIRT_DRIVER" in
        libvirt)
            case "$LIBVIRT_TYPE" in
                lxc) # the cirros root disk in the uec tarball is empty, so it will not work for lxc
                    DEFAULT_IMAGE_NAME=${DEFAULT_IMAGE_NAME:-cirros-${CIRROS_VERSION}-${CIRROS_ARCH}-rootfs}
                    DEFAULT_IMAGE_FILE_NAME=${DEFAULT_IMAGE_FILE_NAME:-cirros-${CIRROS_VERSION}-${CIRROS_ARCH}-rootfs.img.gz}
                    IMAGE_URLS+="http://download.cirros-cloud.net/${CIRROS_VERSION}/${DEFAULT_IMAGE_FILE_NAME}";;
                *) # otherwise, use the qcow image
                    DEFAULT_IMAGE_NAME=${DEFAULT_IMAGE_NAME:-cirros-${CIRROS_VERSION}-${CIRROS_ARCH}-disk}
                    DEFAULT_IMAGE_FILE_NAME=${DEFAULT_IMAGE_FILE_NAME:-cirros-${CIRROS_VERSION}-${CIRROS_ARCH}-disk.img}
                    IMAGE_URLS+="http://download.cirros-cloud.net/${CIRROS_VERSION}/${DEFAULT_IMAGE_FILE_NAME}";;
                esac
            ;;
        vsphere)
            DEFAULT_IMAGE_NAME=${DEFAULT_IMAGE_NAME:-cirros-0.3.2-i386-disk.vmdk}
            DEFAULT_IMAGE_FILE_NAME=${DEFAULT_IMAGE_FILE_NAME:-$DEFAULT_IMAGE_NAME}
            IMAGE_URLS+="http://partnerweb.vmware.com/programs/vmdkimage/${DEFAULT_IMAGE_FILE_NAME}";;
        xenserver)
            DEFAULT_IMAGE_NAME=${DEFAULT_IMAGE_NAME:-cirros-0.3.5-x86_64-disk}
            DEFAULT_IMAGE_FILE_NAME=${DEFAULT_IMAGE_NAME:-cirros-0.3.5-x86_64-disk.vhd.tgz}
            IMAGE_URLS+="http://ca.downloads.xensource.com/OpenStack/cirros-0.3.5-x86_64-disk.vhd.tgz"
            IMAGE_URLS+=",http://download.cirros-cloud.net/${CIRROS_VERSION}/cirros-${CIRROS_VERSION}-x86_64-uec.tar.gz";;
    esac
    DOWNLOAD_DEFAULT_IMAGES=False
fi
```

请下载到内网，并准备访问方式。这里使用 https://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-s390x-disk1.img 镜像。

### openstack源码
由于`devstack`安装`openstack`模式，默认使用从`git`源安装的方式，但是，所有的客户端都直接从`pypi`源安装，如果客户端需要使用源码安装最新版本，可以通过配置的方式实现。那么需要把所有需要的源码同步到内网的`git`库，一般使用`gitlab`来托管。
这里梳理了`devstack`默认需要的模块，如果需要增加，请自行处理。这里修改了一个工具来达到同步`git`源码的目的。参考 [gitsync.sh](gitsync.sh) ,配置项 [list](list).执行同步操作

> gitsync.sh list


```
cinder
glance
heat
horizon
ironic
keystone
neutron
neutron-fwaas
neutron-lbaas
neutron-vpnaas
nova
swift
requirements
tempest
tempest-lib
python-cinderclient
python-glanceclient
python-heatclient
python-ironicclient
keystoneauth
python-keystoneclient
python-neutronclient
python-novaclient
python-swiftclient
python-openstackclient
cliff
futurist
debtcollector
automaton
oslo.cache
oslo.concurrency
oslo.config
oslo.context
oslo.db
oslo.i18n
oslo.log
oslo.messaging
oslo.middleware
oslo.policy
oslo.reports
oslo.rootwrap
oslo.serialization
oslo.service
oslo.utils
oslo.versionedobjects
oslo.vmware
pycadf
stevedore
taskflow
tooz
pbr
glance_store
heat-cfntools
heat-templates
django_openstack_auth
keystonemiddleware
swift3
ceilometermiddleware
os-brick
ironic-lib
dib-utils
os-apply-config
os-collect-config
os-refresh-config
ironic-python-agent
noVNC
devstack
```

## 配置

## 目标机器的系统源配置
参考：
http://mirrors.aliyun.com/help/centos
http://mirrors.aliyun.com/help/epel

## pip源配置

* 注意
安装过程发现devstack有时使用 `sudo` 运行 `pip` ,有时又没有，所以这里需要在用户 `stack` 和 `root` 都配置 pip 源。且非 https 需要加入如下内容： 

```
[global]
index-url = http://mirrors.aliyun.com/pypi/simple/

[install]
trusted-host=mirrors.aliyun.com
[download]
trusted-host=mirrors.aliyun.com
```

参考：
http://mirrors.aliyun.com/help/pypi

## devstack配置

详细使用请参考手册，这里只要描述如何切换到 `intranet` 的环境。

### 修改配置文件

> git clone https://localhost/devstack.git

> cd devstack 

> cp samples/local.conf .

> vi local.conf

在 local.conf 文件中，增加如下配置项，并指向内部源。

```
GIT_BASE = http://localhost
NOVNC_REPO = http://localhost/openstack/noVNC.git
SPICE_REPO = http://localhost/openstack/spice-html5.git

PIP_GET_PIP_URL=http://localhost/soft/get-pip.py

# https://localhost/soft/etcd/{version}/etcd-{version}-linux-{ETCD_ARCH}.tar.gz
ETCD_DOWNLOAD_URL=https://localhost/soft/etcd

UPPER_CONSTRAINTS_FILE=https://github.com/openstack/requirements/plain/upper-constraints.txt

# 默认的为外部网络，disable 它
DOWNLOAD_DEFAULT_IMAGES = false

IMAGE_URLS="http://localhost/images/xenial-server-cloudimg-s390x-disk1.img"

SKIP_EPEL_INSTALL=true

```

### 设置环境变量

由于 `tempest` 在运行时，读取不到 `local.conf` 的信息，所以在运行 `devstack` 前，需要设置其需要的环境变量

> export UPPER_CONSTRAINTS_FILE=https://github.com/openstack/requirements/plain/upper-constraints.txt


### 运行

> ./stack.sh












