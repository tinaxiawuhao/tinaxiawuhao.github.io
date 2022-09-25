---
title: 'Linux文件操作高频使用命令'
date: 2021-04-25 10:28:57
tags: [linux]
published: true
hideInList: false
feature: /post-images/yzriNkrRn.png
isTop: false
---
## 新建操作：

```shell
mkdir abc #新建一个文件夹
touch abc.sh #新建一个文件
```

## 查看操作

###  查看目录：

```shell
ll #显示目录文件详细信息 
du -h 文件/目录  #查看大小
pwd #显示路径
```

### 查看文件内容：

#### cat|head|tail命令

```shell
#查看abc的内容 
cat abc.txt      
 #查看abc前5行内容。默认是10行 
head -5 abc.txt 
tail [选项] 文件名 各选项的含义如下：
    +num：从第num行以后开始显示 
    -num：从距文件尾num行处开始显示。如果省略num参数，系统默认值为10. 
    -f: 循环读取,例如查看服务器日志时，可以实时观察 
#filename 文件里的最尾部的内容显示在屏幕上，并且不断刷新。 
tail -f filename  
#查看最后20行 
tail -f filename
```

#### more命令：

```shell
more命令一次显示一屏信息，若信息未显示完屏幕底部将出现“-More-（xx%）”。

此时按Space键，可显示下一屏内容；

按“回车”键，显示下一行内容；

按B键，显示上一屏；

按Q键，可退出more命令。
```

#### less命令：

和more命令类似，但是比more命令更强大。在很多时候，必须使用less,比如管道。例如：

```shell
ll /etc | less
```

#### stat 命令：

查看文件的详细信息，比如创建修改时间，大小等

```sh
[root@localhost zx]# stat index.html 
  文件："index.html"
  大小：29006     	块：64         IO 块：4096   普通文件
设备：fd00h/64768d	Inode：17589607    硬链接：1
权限：(0644/-rw-r--r--)  Uid：(    0/    root)   Gid：(    0/    root)
环境：unconfined_u:object_r:home_root_t:s0
最近访问：2021-04-25 21:47:41.824053666 +0800
最近更改：2021-04-25 21:44:33.588587500 +0800
最近改动：2021-04-25 21:44:33.588587500 +0800
创建时间：-
```

#### du 命令：

选项：-h 以合适的单位显示（会根据文件的大小自动选择kb或M等单位）

```shell
[root@localhost zx]# du -h index.html
32K	index.html
```



## 复制操作

### 同一机器的复制：

#### cp:复制文件或目录

#### 语法：

```shell
cp [options] source dest

    -a：此选项通常在复制目录时使用，它保留链接、文件属性，并复制目录下的所有内容。其作用等于dpR参数组合。
    -d：复制时保留链接。这里所说的链接相当于Windows系统中的快捷方式。
    -f：覆盖已经存在的目标文件而不给出提示。
    -i：与-f选项相反，在覆盖目标文件之前给出提示，要求用户确认是否覆盖，回答"y"时目标文件将被覆盖。
    -p：除复制文件的内容外，还把修改时间和访问权限也复制到新文件中。
    -r：若给出的源文件是一个目录文件，此时将复制该目录下所有的子目录和文件。
    -l：不复制文件，只是生成链接文件。
```

#### 举例：

```shell
#将../html/index.html 复制到当前目录
cp ../html/index.html .    
#将../html/ 目录下的文件及子目录复制到当前的tt目录下，如果tt不存在，会自动创建
cp -r ../html/  tt/       
#将文件file复制到目录/usr/men/tmp下，并改名为file1
cp file /usr/men/tmp/file1  
    
#如果dir2目录已存在，则需要使用
cp -r dir1/. dir2
#如果这时使用cp -r dir1 dir2,则也会将dir1目录复制到dir2中，明显不符合要求。
ps:dir1、dir2改成对应的目录路径即可
```



### 远程复制

```shell
#将当前目录下的test.txt复制到远程111.12机器的/zx目录下
scp test.txt root@192.168.111.12:/zx

#将test.txt复制到远程用户的根目录，并命名为textA.txt
scp test.txt root@192.168.111.12:testA.txt

#也可以不指定用户，在后续提示中再输入，如下：
scp test.txt 192.168.111.12:/zx

#从远程复制到本地： -r用于递归整个目录
scp -r remote_user@remote_ip:remote_folder local_path
```

##  移动操作:

移动操作可以理解成复制文件后，删除原文件。

```shell
#复制/zx/soft目录中的所有文件到当前目录
mv /zx/soft/* . 
#复制当前目录a.txt到当前的test目录下。
mv a.txt ./test/a.txt  
#复制文件夹到/tmp/下，必须保证tmp是存在的文件夹
mv /zx/soft/  /tmp/soft  
```



## 重命名操作

重命名还是用的移动操作命令，比如：

```shell
#将目录(文件)A重命名为B
mv A B
#将/a目录(文件)移动到/b下，并重命名为c。要保证b目录存在。
mv /a /b/c
#将当前test1目录移动到当前的test目录并命名为b
mv ./test1 ./test/b 
```



## 解压压缩操作

### tar

```shell
-c: 建立压缩档案
-x：解压
-t：查看内容
-r：向压缩归档文件末尾追加文件
-u：更新原压缩包中的文件
这五个是独立的命令，压缩解压都要用到其中一个，可以和别的命令连用但只能用其中一个。下面的参数是根据需要在压缩或解压档案时可选的。

-z：有gzip属性的
-j：有bz2属性的
-Z：有compress属性的
-v：显示所有过程
-O：将文件解开到标准输出

下面的参数-f是必须的
-f: 使用档案名字，切记，这个参数是最后一个参数，后面只能接档案名。
```

#### 举例说明

```shell
tar -cf all.tar *.jpg
这条命令是将所有.jpg的文件打成一个名为all.tar的包。-c是表示产生新的包，-f指定包的文件名。
tar -tf all.tar
这条命令是列出all.tar包中所有文件，-t是列出文件的意思
tar -xf all.tar
这条命令是解出all.tar包中所有文件，-x是解开的意思
压缩
tar –cvf jpg.tar *.jpg //将目录里所有jpg文件打包成jpg.tar
eg2:
    tar -xzf nginx-1.14.0.tar.gz //解压到当前目录
    tar -zxf nginx-1.14.0.tar.gz -C /usr/local/nginx  #解压到对应目录
 eg3:
 	tar -zxvf nginx...tar.gz #解压并显示过程
 	
注意：有些压缩程序提示命令找不到，需要进行安装，例如：
yum install unzip
或在ubuntu上：
apt-get install unzip
```

#### 总结

```shell
1、*.tar 用 tar –xvf 解压
2、*.gz 用 gzip -d或者gunzip 解压
3、*.tar.gz和*.tgz 用 tar –xzf 解压
4、*.bz2 用 bzip2 -d或者用bunzip2 解压
5、*.tar.bz2用tar –xjf 解压
6、*.Z 用 uncompress 解压
7、*.tar.Z 用tar –xZf 解压
8、*.rar 用 unrar e解压
9、*.zip 用 unzip 解压  

解压的时候，有时候不想覆盖已经存在的文件，那么可以加上-n参数
unzip -n test.zip
unzip -n -d /temp test.zip
只看一下zip压缩包中包含哪些文件，不进行解压缩
unzip -l test.zip
查看显示的文件列表还包含压缩比率
unzip -v test.zip
检查zip文件是否损坏
unzip -t test.zip
如果已有相同的文件存在，要求unzip命令覆盖原先的文件
unzip -o test.zip -d /tmp/
示例：
 eg1:  unzip mydata.zip -d mydatabak #解压到mydatabak目录
 
1.  xz
	这是两层压缩，外面是xz压缩方式，里层是tar压缩,所以可以分两步实现解压
	$ xz -d node-v6.10.1-linux-x64.tar.xz
    $ tar -xvf node-v6.10.1-linux-x64.tar
```



## 上传文件工具

从本地windows上传一些文件到远程Linux服务器可以通过xshell的xftp也可以通过下面这个小工具lrzsz，使用更加方便。

```shell
#安装工具
yum install lrzsz 

常用命令：
#下载文件dist.zip到本地
sz dist.zip 
#会打开窗口，上传文件到远程服务器
rz 
```



## ln、file和touch命令

###  ln命令

用于创建链接文件，包括硬链接(Hard Link)和符号链接（Symbolic Link) 。我们常用的是符号链接，也称软连接。软连接就类似windows里的快捷方式。

####  示例：

```shell
#在当前目录创建一个软连接，指向/etc/fastab，名称也是fastab
ln -s /etc/fastab

#在当前目录创建一个指向/boot/grub的软连接，命名为gb
ln -s /boot/grub gb
```

注意：删除软连接 正确方式是：

```shell
rm -rf ./gb
```

错误方式:

```shell
rm -rf ./gb/
```

这样会删除了原有grub下的内容。特别是针对系统文件的软连接，删除一定要慎重。

#### file命令

用于识别文件的类型

Linux中文件后缀只是方便使用者识别，没有实质的约束作用。file命令可以查看文件的实质类型：

```shell
file [-bcLz] 文件|目录

选项说明：

    文件|目录：需要识别的文件或目录
    -b: 显示识别结果时，不显示文件名
    -c: 显示执行过程
    -L: 直接显示符号链接文件指向的文件类型
    -z: 尝试去解读压缩文件的内容
```



#### 示例：

 可以看出，index.mp4本质是一个HTML而非一个mp4文件

```shell
[root@VM_0_13_centos soft]# file index.mp4 
index.mp4: HTML document, UTF-8 Unicode text, with very long lines**touch命令：** 用于改变文件或目录的访问时间和修改时间。
```

###  touch命令：

用于改变文件或目录的访问时间和修改时间。

```shell
touch [-am] [-t<日期时间>] [目录|文件]
```

如果指定目录文件不存在，则会直接创建一个空文件，所以touch也常用来创建一个空白文件

```shell
#创建一个新文件aa.txt
touch aa.txt

选项说明：

    -a: 只修改访问时间
    -m : 只修改 修改时间
    -t : 使用指定日期时间，而非系统时间 。例如要修改为2019年10月20日16：38分13秒。参数就是：‘20191020163813’
```

#### 示例：

修改之前可以先查看文件的时间戳: 用stat 命令查看

```shell
[root@VM_0_13_centos soft]# stat index.html 
  File: ‘index.html’
  Size: 17215     	Blocks: 40         IO Block: 4096   regular file
Device: fd01h/64769d	Inode: 529352      Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2021-04-25 15:15:37.280616254 +0800
Modify: 2021-04-25 15:15:37.280616254 +0800
Change: 2021-04-25 15:15:37.290616257 +0800
 Birth: -
```

开始修改：将index.html文件的访问和修改时间修改成当前系统的时间。

```shell
touch index.html
```

## 查找操作命令：

对于要用到的文件，目录等，经常有忘记的时候，所以查找命令就显得极为必要：

### find: 

查找文件或目录 (**常用**)

语法如下：

```shell
find [目录…] [-amin <分钟>] [-atime <24小时数>] [-cmin <分钟>] [-ctime<24小时数>][-empty][-exec<执行命令>][-fls<列表文件>][-follow]
    [-fstype <系统文件类型>] [-gid <组编号>] [-group <组名称>] [-nogroup] [-mmin <分钟>]  [-mtime
    <24小时数>] [-name <查找内容>] [-nogroup] [-nouser] [-perm <权限数值>] [-size <文件大小>] [-uid
    <用户编号>] [-user <用户名称>] [-nouser]
```

**几个常用选项说明：**

```shell
    -size <文件大小>：查找符合指定大小的文件。文件大小单位可以是“c”表示Byte；“k”表示KB。如配置为“100k”，find命令会查找文件大小正好100KB的文件；配置为“+100k”，find命令会查找文件大小大于100KB的文件；配置为“-100k”，find命令会查找文件大小小于100KB的文件。
    -user<用户名称>：查找所有者是指定用户的文件或目录，也能以用户编号指定
    -name <查找内容>：查找指定的内容，在查找内容中使用“*” 表示任意个字符；使用“?”表示任何一个字符
    -mtime <24小时数>：查找在指定时间**曾更改过内容**的文件或目录，单位以24小时计算。如配置为2，find命令会查找刚好在48小时之前更改过内容的文件；配置为+2，find命令会查找超过在48小时之前更改过内容的文件；配置为-2，find命令会查找在48小时之内更改过内容的文件。
    -mmin <分钟>：查找在指定时间曾被**更改过内容**的文件或目录，单位以分钟计算。
    -cmin <分钟>：查找在指定时间曾被**更改过权限**属性的文件或目录，单位以分钟计算。-ctime对应小时。
    -amin <分钟>：查找的是指定时间**访问过**的文件或目录。-atim对应小时。
    -perm <权限数值>：查找符合指定权限数值（有关权限数值见第6章）的文件或目录。如配置为“0700”，find命令会查找权限数值正好是“0700”的文件或目录；配置为“+0700”，find命令会查找权限数值大于 “0700”的文件或目录；配置为“-0700”，find
```

选项大概有以下几类：

1.按时间范围查找

2.按文件大小查找

3.按文件名称查找

4.按其他：比如权限、用户组、类型等

#### 示例：

```shell
#从根目开始，查找名称以nginx开头的目录和文件
find / -name nginx*  

#查找文件大小超过100M的文件
find / -size +100M

#查找/home/zx目录下，10分钟内被修改过的文件和目录
find /home/zx/  -mmin -10
```

### locate：

 查找文件或目录(**不常用**)

```shell
locate 查找内容
```

例如：locate nginx 会将所有包含nginx的目录和文件都列出来。可以用* 或？等匹配符。

locate的查找速度非常快，因为该命令查找的是数据库，所以有些刚修改的文件和目录，可能无法找到。可以采用：updatedb 命令更新数据库。

### which: 

查找文件(**不常用**)

```shell
which [文件]

which命令只会在PATH环境变量定义的路径及命令别名中查找，所以范围有限。
```



### whereis : 

查找文件(**不常用**)

```shell


whichis [-bu] [-B<目录>] [-M<目录>] [-S<目录>] [文件]

常用选项：

    文件：要查找的命令

    -b: 只查找二进制文件

    -u: 查找不包含指定类型的文件

    -B<目录>： 只在指定目录下查找二进制文件

    -M<目录>：只在指定目录查找帮助文件

    -S<目录>：只在指定目录查找源码目录

例如： 默认只会在指定目录查找（/bin ,/etc ,/usr)

[root@VM_0_13_centos soft]# whereis nginx
nginx: /usr/local/nginx /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx.bak
```

