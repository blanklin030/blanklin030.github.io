---
title: 由于overlay占用磁盘空间导致磁盘告警
date: 2022-02-15 15:09:08
tags:
  - linux
categories:
  - sre
---

### 背景
早上突然收到告警短信，xxx服务磁盘使用率超过80%，如下图，赶紧上机器进行排查，下面是排查的过程
![avatar](/images/linux/disk_full/1.png)

### 查看机器整体磁盘使用情况
```
df -h

// 下面是结果
root@xxx.docker.ys:~/dlap-manager$ df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          20G   18G  2.7G  87% /
tmpfs            64M     0   64M   0% /dev
tmpfs            63G     0   63G   0% /sys/fs/cgroup
/dev/sda3       219G   15G  193G   8% /etc/hosts
shm              64M     0   64M   0% /dev/shm
/dev/sdb1       4.4T  949G  3.5T  22% /etc/hostname
tmpfs            63G   12K   63G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs            63G     0   63G   0% /proc/acpi
tmpfs            63G     0   63G   0% /proc/scsi
tmpfs            63G     0   63G   0% /sys/firmware
```
看到overlay这个文件目录占了`87%`，仔细研究下这个文件夹是什么

#### overlay介绍
OverlayFS是一种堆叠文件系统，它依赖并建立在其它的文件系统之上（例如ext4fs和xfs等等），并不直接参与磁盘空间结构的划分，仅仅将原来底层文件系统中不同的目录进行“合并”，然后向用户呈现，这也就是联合挂载技术，对比于AUFS，OverlayFS速度更快，实现更简单。 而Linux 内核为Docker提供的OverlayFS驱动有两种：overlay和overlay2。而overlay2是相对于overlay的一种改进，在inode利用率方面比overlay更有效。但是overlay有环境需求：docker版本17.06.02+，宿主机文件系统需要是ext4或xfs格式。
overlayfs通过三个目录：lower目录、upper目录、以及work目录实现，其中lower目录可以是多个，work目录为工作基础目录，挂载后内容会被清空，且在使用过程中其内容用户不可见，最后联合挂载完成给用户呈现的统一视图称为为merged目录
![avatar](/images/linux/disk_full/2.png)
在上述图中可以看到三个层结构，即：lowerdir、uperdir、merged，其中lowerdir是只读的image layer，其实就是rootfs，对应的lowerdir是可以有多个目录。而upperdir则是在lowerdir之上的一层，这层是读写层，在启动一个容器时候会进行创建，所有的对容器数据更改都发生在这里层。最后merged目录是容器的挂载点，也就是给用户暴露的统一视角。

#### overlay如何工作
当容器中发生数据修改时候overlayfs存储驱动又是如何进行工作的？以下将阐述其读写过程：
 
+ 1. 读：
  + 如果文件在容器层（upperdir），直接读取文件；
  + 如果文件不在容器层（upperdir），则从镜像层（lowerdir）读取；
+ 2. 写：
  + 首次写入： 如果在upperdir中不存在，overlay和overlay2执行copy_up操作，把文件从lowdir拷贝到upperdir，由于overlayfs是文件级别的（即使文件只有很少的一点修改，也会产生的copy_up的行为），后续对同一文件的在此写入操作将对已经复制到容器的文件的副本进行操作。这也就是常常说的写时复制（copy-on-write）
  + 删除文件和目录： 当文件在容器被删除时，在容器层（upperdir）创建whiteout文件，镜像层(lowerdir)的文件是不会被删除的，因为他们是只读的，但without文件会阻止他们显示，当目录在容器内被删除时，在容器层（upperdir）一个不透明的目录，这个和上面whiteout原理一样，阻止用户继续访问，即便镜像层仍然存在。 
+ 3. 注意事项
  + copy_up操作只发生在文件首次写入，以后都是只修改副本,
  + overlayfs只适用两层目录，相比于比AUFS，查找搜索都更快。
  + 容器层的文件删除只是一个“障眼法”，是靠whiteout文件将其遮挡，image层并没有删除，这也就是为什么使用docker commit 提交保存的镜像会越来越大，无论在容器层怎么删除数据，image层都不会改变。

#### overlay镜像存储结构
从仓库拉取ubuntu镜像，结果显示总共拉取了6层镜像如下：
```
[hadoop@bigdata-xxx.ys ~]$ sudo docker pull ubuntu-test:xenial
Trying to pull repository ubuntu-test ...
xenial: Pulling from ubuntu-test
58690f9b18fc: Pull complete
b51569e7c507: Pull complete
da8ef40b9eca: Pull complete
fb15d46c38dc: Pull complete
4c7a0de79adc: Pull complete
5eff12cba838: Pull complete
Digest: sha256:d21c70f1203a5b0fe1d8a1b60bd1924ca5458ba450828f6b88c4e973db84c8e8
Status: Downloaded newer image for ubuntu-test:xenial
```
此时6层镜像被存储在/var/lib/docker/overlay2下，将镜像运行起来一个容器，如下命令
```
sudo docker run -itd --name ubuntu-test ubuntu-test:xenial
```
运行容器之后，查询出容器的元数据
```
// 获取容器ID
sudo docker ps -a | grep ubuntu
// 通过容器ID查看元数据
sudo docker inspect container-id
```
![avatar](/images/linux/disk_full/4.png)

进入容器创建一个文件，如下命令
```
sudo docker exec -it ubuntu-test bash
echo 'xxxxxx' > helloworld.txt
```

通过tree命令查看新创建的文件出现在哪里
```
tree -L 3 /var/lib/docker/overlay2/f90282cedadf6b7765ba06ea8983af306f78773baeca9ff1e418651d0b9d14f2/diff
```
![avatar](/images/linux/disk_full/3.png)

### 回到问题本身
overlay目录使用了87%的存储，触发了磁盘告警，overlay目录是我们的弹性云容器运行的目录，发现通过nohup启动的jar包将所有stdout/stderr都输出到了nohup.out文件，该文件也持久化存在/var/lib/docker/overlay2下，所以需要对nohup.out进行处理，不可直接删除，因为jar启动后所有的输出流都会转存到nohup.out，这个文件句柄已经打开，所有的stdout/stderr仍然会输出，只是输出在/proc/pid/fd/1或者/proc/pid/fd/2这些目录下。
> 0描述符 标准输入
> 1描述符 标注输出
> 2描述符 标注错误输出  

**linux一切到可以看作文件**，/proc/pid/fd/1 就是pid进程的标准输出，/proc/pid/fd/1 就是pid进程的标准错误输出，我们通过测试可以看到下面结果，就算我们删除了nohup.out文件，任何会将stdout/stderro输出到这个文件。
```
[hadoop@xxx.ys ~]$ tree -L 3 /proc/143645/fd
/proc/143645/fd
├── 0 -> /dev/null
├── 1 -> /home/hadoop/nohup.out\ (deleted)
├── 2 -> /home/hadoop/nohup.out\ (deleted)
└── 3 -> /home/hadoop/test.txt

0 directories, 4 files
```
+ 1. 重启jar包，将所有错误/标准输出忽略
```
// >/dev/null是表示linux的空设备，是一个特殊的文件，写入到它的内容都会被丢弃(等同于 1>/dev/null)
// 2>&1表示所有错误输出都重定向到标准输出
// 所以就是将错误输出重定向到标准输出，将标准输出重定向到空设备，就是禁止所有输出/错误
nohup java -jar xx.jar >/dev/null 2>&1 &
```
+ 2. 删除nohup.out文件
```
rm -f nohup.out
```

### 修改docker的根目录
#### 安装过程
1. 停止docker服务
```
systemctl stop docker
```
2. 创建新的docker工作目录
```
mkdir -p /data2/docker
```
这个目录可以自定义，但是一定要保证在/root里面

3. 迁移/var/lib/docker
```
rsync -avz /var/lib/docker /data2/docker/
```
4. 配置devicemapper.conf
```
# 不存在就创建
sudo mkdir -p /etc/systemd/system/docker.service.d/
# 不存在就创建
sudo vi /etc/systemd/system/docker.service.d/devicemapper.conf
```
5. 在devicemapper.conf中添加
```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd  --graph=/data2/docker
```

6. 重启docker服务
```
systemctl daemon-reload
 
systemctl restart docker
 
systemctl enable docker
```

7. 确认是否配置成功
```
docker info
```
![avatar](/images/linux/disk_full/5.png)

重新启动所有容器后，确认无误。即可删除/var/lib/docker里面所有文件。
#### 注意事项
如果报错如下
```
Error response from daemon: Cannot restart container linyu: shim error: docker-runc not installed on system
```
+ 1. 判断是否安装runc
```
 rpm -qi runc
```
+ 2. 未安装则先安装
```
sudo yum install runc
```
+ 3. 创建软连接
```
cd /usr/libexec/docker/
sudo ln -s docker-runc-current docker-runc 
```
+ 4. 重启服务
```
sudo docker restart container_id
```
