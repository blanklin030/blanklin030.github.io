---
title: 在centos7下安装python3
date: 2020-05-19 18:33:10
tags:
  - python
categories:
  - tools
---
> 在https://www.python.org/ftp/python/中选择自己需要的python源码包，我下载的是python3.7.0  

+ 下载
```
wget https://www.python.org/ftp/python/3.7.0/Python-3.7.0.tgz
```
+ 解压
```
tar zxvf Python-3.7.0.tgz
```
+ 安装
```
cd Python-3.7.0
./configure --prefix=/usr/local/python3
make && make install
```
> 安装完成没有提示错误便安装成功了  

+ 建立软连接
```
ln -s /usr/local/python3/bin/python3.7 /usr/bin/python3
ln -s /usr/local/python3/bin/pip3.7 /usr/bin/pip3
```
+ 测试
```
python3 --version
```

+ 卸载python
```
sudo rm -fr /usr/local/python3
sudo rm -fr /usr/bin/python3
sudo rm -fr /usr/bin/pip3
```

+ ssl错误
> Could not fetch URL https://pypi.org/simple/pip/: There was a problem confirming the ssl certificate: HTTPSConnectionPool(host='pypi.org', port=443): Max retries exceeded with url: /simple/pip/ (Caused by SSLError("Can't connect to HTTPS URL because the SSL module is not available.")) - skipping


1. 安装openssl
```
yum -y install openssl
```
2. 卸载python和pip

3. 重新安装python

+ 升级pip
```
curl https://bootstrap.pypa.io/get-pip.py | python
```