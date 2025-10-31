# 磁盘配额

默认情况容器内的可用磁盘空间是没有限制的。在容器内往容器文件系统里写东西（任何目录都没有挂载任何外部存储卷），此时写入的数据，都写到了 upperdir层，也就是写到的宿主机上。如果不加以限制，很有可能会发宿主机磁盘空间写满。

这个问题有两种解决方式：

- 对容器的可用磁盘做配额。
- 容器写操作的目录，挂载一个专门的外部存储卷（推荐）。



#### 单独一个容器做磁盘配额

创建一个容器，单独配额，使用 docker run 的选项 ` --storage-opt` 指定配额大小。

~~~bash
# 磁盘配额 100M
docker run -d  --storage-opt size=100M centos:7 tail -f /dev/null
~~~

```alert type=note
`--storage-opt` 是依赖 xfs 的 quota 功能的，所以以该参数启动容器前，必须为 docker 所在数据目录的磁盘 remount 一个 pquota 属性，否则以该参数启动容器会报错。
```



#### 设置全局的默认值

配置在 `/etc/docker/daemon.json` ，配置保存后，重启 docker 服务。后面创建的容器自动使用默认磁盘配额。

~~~json
# 将每个容器可以使用的磁盘空间设置为 1G
{
    "storage-opts": [
        "overlay2.override_kernel_check=true",
		 "overlay2.size=1G"
	]
}
~~~



~~~bash
# 1、针对xfs的quota限额的使用有3项：
  （1）. usrquota:针对使用者账号
  （2）. grpquota:针对群组
  （3）. prjquota:针对单一目录，但是不能与grpquota同时存在
   ps：ext4在2016年开始支持project quota
 
# 2、--storage-opt是依赖xfs的quota功能的，所以以该参数启动容器前，必须为docker所在数据目录的磁盘remount一个pquota属性，否则以该参数启动容器会报错
docker: Error response from daemon: --storage-opt is supported only for overlay over xfs with 'pquota' mount option.
 
如果你的数据目录是一个新磁盘，那么直接：mount -o remount,prjquota /dev/sda3 即可
如果是已经在使用的磁盘，需要修改/etc/fstab,加上prjquota属性
/dev/sdb /home    xfs    defaults,prjquota    0 0
 
然后修改下述内核参数，重启主机
 
# 3、修改内核参数
~~~



