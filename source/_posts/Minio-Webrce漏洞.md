title: Minio Webrce漏洞
author: 几时西风
tags:
  - 安全
  - minio
categories:
  - 安全
date: 2023-05-03 10:54:00
---
# Minio web应用命令执行（Webrce）漏洞
20230502，五一期间，人在家中坐，祸从天上来。

## 背景
公司上任架构师在阿里云搭建了包含三个节点的Minio集群作为OSS服务，接手后基本就是巡查一下然后定期更新版本，偶尔磁盘空间不足的话扩容一下，一切都是那么平稳且无趣。

## 问题发现
1. 20230502 11:09:33，阿里云云安全中心发出安全告警，提示“Web应用命令执行，异常网络流量”。具体如下图

![upload successful](\blog\images\pasted-29.png)

![upload successful](\blog\images\pasted-30.png)

## 问题分析
从上面第二张图可以看到，攻击方式非常的朴实无华，攻击方利用python的requests包向minio集群发起了一个特殊的GET请求，然后居然就能通过minio这个父进程调用shell命令，直接执行了whoami这个命令。

乍见感到十分惊讶，这样简单的方式居然就能攻破minio集群，网上搜索一番，没有相关资料，尤其是minio源码或者文档里完全没有提及有这么个请求地址 **<font color="red">http://{minio地址}:{minio端口}/?alive9c={shell命令}</font>**。

遂决定自己试试，以下是随手编写的python代码
```python
import requests
r=requests.get("http://{minio地址}:{minio端口}/?alive9c=pwd")
print(r.content)
```
结果令人惊讶，居然真的就获得了minio的路径，而且通过这个方式基本可以执行minio用户下所有有权限执行的shell命令，如果安装minio时用户名密码是通过环境变量或者配置文件设置的话，很简单的shell命令就可以获取到minio的管理账号。

## 问题解决
刚发现问题时，第一步其实是阻断攻击者的ip访问，到云服务器ECS的网络安全组中，先设置访问规则，将所有来自该ip的访问阻断。

然后就开始搜索相关资料，试图找到相关信息看如何针对性地修补该漏洞。

遗憾的是，minio源码或者文档里完全没有提及有这个导致问题的请求地址是做什么的，思考一番决定尝试更新minio版本解决此问题。

利用下面的命令，先查看minio集群目前的版本，然后尝试更新：
```bash
mc admin info myminio
mc admin update myminio
```
发现问题："Server `myminio` already running the most recent version of MinIO"

再看minio版本居然是development，具体如下图：

![upload successful](\blog\images\pasted-31.png)

具体为何会更新至这个版本已经不可追溯，但是minio集群因为这个问题会一直停滞在这个development版本而不更新，怀疑出问题的地址就是仅用于开发或者之前某个版本被人恶意下毒，然后在利用mc admin update 时更新到了这个有问题的版本，然后就被利用了这个漏洞。

于是，手动更新mc(客户端)与minio（服务端）：
1. 获取官方源上最新版本的二进制文件：

```bash
wget https://dl.min.io/server/minio/release/linux-amd64/minio
wget https://dl.min.io/client/mc/release/linux-amd64/mc

```
2. 停止minio服务，并备份原二进制文件：

```bash
systemctl stop minio
cd {minio安装目录}/bin
mv minio minio.bak
mv mc mc.bak
```
3.将下载好的文件放到minio的bin目录下替换原有二进制文件
4.修改文件属性(组、所有者、可执行):

```bash
chmod a+x minio
chmod a+x mc
chown minio.minio minio
```
5.重启minio并观察服务状态:
```bash
systemctl start minio
systemctl status minio
```

**最后再次利用之前的python脚本请求有问题的地址，发现已经不可执行shell命令了，此问题解决。**

## 经验总结
1. 阿里云的云安全服务是个好东西，这个钱不能省；
2. 开源的东西问题多，其实建议换成熟云服务商提供的OSS服务，不要自己折腾；
3. 有的更新可能会导致问题，不要瞎更新；
4. 发现攻击可以先阻断攻击路径，切他网线和ip，后面再慢慢解决漏洞；
5. 还好运行minio的权限不是root，再次警示不要所有服务都运行在root账号上。

## Todo
都提到了minio，后续有空写一下如何搭建minio集群与如何为minio集群动态扩容，还有一些minio的使用注意事项与demo