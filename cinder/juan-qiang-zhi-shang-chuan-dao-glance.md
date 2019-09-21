# 卷强制上传到glance

#### 问题

将正在使用中的云磁盘上传到glance中时，即使使用强制模式（Web管理页面中勾选了强制选择、命令行有`--force`参数），依然无法成功。报错信息为：磁盘正在使用中, `Volume status is in-use`。

#### 原因

由于正在使用中的磁盘会有数据不断写入，而且存储底层的驱动类型非常多，不一定支持热备份。因此cinder服务默认关闭了强制上传到镜像服务的选项，但有时候有这样的需求。

**解决方案**

修改cinder配置文件`/etc/cinder/cinder.conf`

```text
enable_force_upload = true
```

重启服务

```text
service cinder-volume restart
service cinder-scheduler restart
service cinder-api restart
```

将指定的磁盘上传到glance

```text
cinder  upload-to-image  2d222555-9990-407d-a9b2-173743fce49c centos7-cloud-181108 --force=True --disk-format=qcow2
```

将镜像保存到本地

```text
openstack image save 13b920ad-a1e5-41aa-9714-635ae6d44741 --file centos7-cloud-181111.qcow2
```

#### 相关链接

[https://bugs.launchpad.net/cinder/+bug/1523230](https://bugs.launchpad.net/cinder/+bug/1523230)

