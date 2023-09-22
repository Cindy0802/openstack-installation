# Ubuntu22.04使用devstack部署OpenStack（All-In-One)

​	<font color="gray">初学OpenStack，配置了快一周，起初用Ubuntu18.04尝试n次，大概因为OpenStack已更新到最新的分支，每次执行安装脚本总有各种依赖冲突、版本不匹配等等的报错。遂换用Ubuntu22.04，~~什么叫纵享丝滑！~~安装部署的过程中再没有上述问题，可能有时候会因为网络问题卡住一下，解决一些小问题，但是整体过程相对于灰头土脸的前半周来说，太顺利辣！遂记录。</font>

## 0.虚拟机配置

#### 清华镜像链接：https://mirrors.tuna.tsinghua.edu.cn/ubuntu-releases/22.04/

<center>this one:</center>

![image-20230922102343294](C:\Users\Cindy\AppData\Roaming\Typora\typora-user-images\image-20230922102343294.png)

### 内存：8GB

### 硬盘：50GB

一开始硬盘跟着默认设的20GB，安装到一半磁盘容量不足，蛮麻烦的。最好一开始就找个容量宽裕一点的磁盘，分配多一点存储空间。

![image-20230922105519208](C:\Users\Cindy\AppData\Roaming\Typora\typora-user-images\image-20230922105519208.png)

## 1.部署前准备

#### 设置root密码

```
sudo passwd root
```

#### 安装vim

```
sudo apt install vim
```

#### 安装net-tools

<font color="gray">为了能使用ifconfig查看本机ip</font>

```
sudo apt install net-tools
```

#### 安装git

```
sudo apt install git
```



#### pip换源

```
mkdir ~/.pip 
sudo vim ~/.pip/pip.conf
```

填入以下内容：

```
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
index-index-url = https://mirrors.aliyun.com/pypi/simple/ 
[install]
trusted-host =
    pypi.tuna.tsinghua.edu.cn
    mirrors.aliyun.com
```

## 2.配置静态IP

```
ifconfig #查看一下本机IP先
```

编辑网卡配置文件

```
chmod a+w  /etc/netplan/01-network-manager-all.yaml #添加写权限
vim /etc/netplan/01-network-manager-all.yaml
```

填入以下内容，要注意2个点：

<mark>①缩进要正确，冒号后有空格</mark>

<mark>②三个填IP地址的地方：第一个填之前ifconfig查到的IP，第二个、第三个填同一网段的IP，不能填跟前面一样的！</mark>

```
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    ens33:
       addresses: [192.168.xxx.yyy/24] #填之前ifconfig查到的IP
       dhcp4: no
       dhcp6: no
       routes: 
          - to: default 
            via: 192.168.xxx.2
       nameservers:
         addresses: [192.168.xxx.2,8.8.8.8]
```

```
#使配置生效
netplan apply
```

这时候可以ping一下baidu.com或者通过虚拟机的浏览器访问一下网页，看是否能正常访问，能访问就是配置成功了！

## 3.devstack部署

#### 创建stack用户

```
useradd -s /bin/bash -d /opt/stack -m stack
chmod +x /opt/stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo -u stack -i
```

#### 克隆devstack

```
git clone https://opendev.org/openstack/devstack
```

#### 新建local.conf

```
cd devstack
vim local.conf
```

填入以下内容：

```
[[local|localrc]]
ADMIN_PASSWORD=密码
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

# host为自己刚刚设置的静态ip 
HOST_IP=192.168.xxx.yyy
```

## 4.开始安装！

由于github连接时常不稳定，推荐先在这把该下的包下了。可能会连接失败，多试两次就行

```
cd files/
wget -c https://github.com/coreos/etcd/releases/download/v3.4.27/etcd-v3.4.27-linux-amd64.tar.gz
```



执行./stack.sh，漫长的过程开始了！！~~也就好几个小时吧~~

```
./stack.sh
```

## 5.可能遇到的问题&解决方案：

①几个地方会卡很久

→正常，等等吧

②Error while executing command: HttpException: 503, Unable to create the networkNo tenant network is available for allocation.

```
./clean.sh
./unstack.sh
```

> DevStack 的 `clean.sh` 脚本通常用于清理 DevStack 安装后的环境，以便重新运行或卸载 DevStack。执行 `clean.sh` 会删除 DevStack 创建的临时文件、虚拟机、网络配置等，但不会删除已下载的软件包。
>
> DevStack 的 `unstack.sh` 脚本通常用于停止所有正在运行的OpenStack服务，并清理掉相关的进程和资源。

<font color="gray">	清理前用ifconfig看的时候，多了一个网络，应该就是之前几次跑的时候openstack分配的虚拟网络。然后第二次跑才会说网络资源不足以分配。清理完之后再看那个虚拟网络就消失了，就可以重新开始运行了！</font>

重新开始运行

```
./stack.sh
```



其他的除了偶尔网络抽风，重新执行就行，好像也没碰到什么问题了。

## 6.SUCCESS！

openstack的登录地址，以及用户名和密码看这里↓

默认有个admin用户，密码就是前面在local.conf配置时填的

![image-20230922130650874](C:\Users\Cindy\AppData\Roaming\Typora\typora-user-images\image-20230922130650874.png)

![image-20230922130713185](C:\Users\Cindy\AppData\Roaming\Typora\typora-user-images\image-20230922130713185.png)

最后记得执行`./unstack.sh`停止服务。

