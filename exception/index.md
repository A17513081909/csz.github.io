---
layout: page
title: '异常处理'
tags: [exception]
date: 2022-06-21
comments: false
---

* ModuleNotFoundError: No module named 'psycopg2._psycopg'    
```
pip install psycopg2
pip install psycopg2-binary 
sudo apt install libpq-dev
sudo apt install python-psycopg2* python3-psycopg2*  
sudo apt install python-dev python3-dev python3.7-dev 
sudo apt install python-setuptools 
```
* pip换源，清华大学源    
```
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```
* Error saving credentials: error storing credentials - err: no credentials server URL, out: no credentials server URL while login into docker 
```  
sudo apt install -y gnupg2 pass
```
* Fatal error When use the Netplan Apply Command on Ubuntu  
```
apt remove netplan  
apt install netplan.io  
```
* ubuntu18.04，dpkg-warning：dpkg: warning: files list file for package 'fonts-sil-abyssinica' missing; assuming package has no files currently installed  
```
将所有warning包拷贝到dpkg-warning.txt文件里  
新建脚本dpkg-pkgreinstall：  
#!/bin/bash  
for package in $(cat dpkg-warning.txt | grep "dpkg: warning: files list file for package " | grep -Po "'[^']*'" | sed "s/'//g")；  
do  
  sudo apt-get install --reinstall "$package" -y;  
done  
给权限，执行即可  
```
* npm ERR! Cannot read properties of null (reading 'pickAlgorithm')
```
npm cache clear --force
npm install
```
* npm ERR! Maximum call stack size exceeded
```
npm -v
npm install -g npm
```

