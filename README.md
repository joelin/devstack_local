# 企业内部网络部署devstack
由于安全原因，企业内部无法访问互联网,有时也由于互联网出口的有限，把多人协作需要频繁访问的互联网资源同步到企业内网。面对这样的环境，如何在内部网络部署最新的openstack版本？此文介绍如何实现。

## 前提条件
熟读devstack的安装手册，了解到devstack安装，除了openstack的所有源码模块外，还需要操作系统源，pip源，镜像文件下载地址。
为了在企业内网安装openstack，需要把这些前提条件都准备，让其在企业内网可用。

### 操作系统源
以　CENTOS 为例，需要准备　CENTOS base源、epel 源，一般使用rsync的方式进行同步。参考链接:    https://wiki.centos.org/zh/HowTos/CreatePublicMirrors?highlight=%28rsync%29

准备完成应该可以提供如下访问方式：　

https://mirrors.aliyun.com/centos/7.3.1611/
https://mirrors.aliyun.com/epel/7/


### pip源
pypi源搭建可以选择多种方式，比如　pypi-mirror　有选择性的同步需要的模块，　rsync、bandersnatch　同步所有的模块。

准备完毕后，应该可以提供如下访问方式：　
https://mirrors.aliyun.com/pypi


### 镜像地址
下载devstack里需要使用的镜像文件，到内网地址，一般由用户自己选择需要使用的镜像
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
由于devstack安装openstack模式，默认使用从git源安装的方式，但是，所有的客户端都直接从pypi源安装，如果客户端需要使用源码安装最新版本，可以通过配置的方式实现。那么需要把所有需要的源码同步到内网的git库，一般使用gitlab来托管。
这里梳理了devstack默认需要的模块，如果需要增加，请自行处理。这里修改了一个工具来达到同步git源码的目的。参考 [gitsync.sh](gitsync.sh) ,配置项 [list](list).执行同步操作

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

参考：
http://mirrors.aliyun.com/help/pypi

## devstack配置
详细使用请参考手册，这里只要描述如何切换到 intranet 的环境。
在 local.conf 文件中，增加如下配置项，并指向内部源。

```
GIT_BASE = http://localhost
NOVNC_REPO = http://localhost/openstack/noVNC.git
SPICE_REPO = http://localhost/openstack/spice-html5.git

UPPER_CONSTRAINTS_FILE=https://github.com/openstack/requirements/plain/upper-constraints.txt


IMAGE_URLS="http://localhost/images/xenial-server-cloudimg-s390x-disk1.img"

```












