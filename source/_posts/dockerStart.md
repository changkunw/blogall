---
title: docker 初级
date: 2016-07-26 18:51:09
tags: docker
---

***为什么要使用docker***

我认为理解docker最好的解释方式就是``轻量级的虚拟机``，很可能大家都会这么理解，这也是最容易的理解方式。那么我们为啥要使用docker呢，我觉得最大的好处就是它能保持垮环境的一致性，启动、停止都是很快，使用很方便，不需要担心开发环境对你造成影响。这只是目前使用后觉得它的好处。

***理解***

在使用之前，我也感觉docker这个东西很神秘，感觉很高大上，但是呢确实应该还是比较高大上，这里我不知道大家会不会有和我有一样的错误理解，就是比如我要搞个jdk的容器，你会不会以为就相当于把jdk按照docker的方式打了个包？，然后用的时候又把这个打好的包下下来你就可以用了？我擦，其实说是也是这样，但是不是那么简单，jdk是要打包，但是jdk运行的时候是需要环境的，所以你要打包jdk你还需要一个操作系统，so，你打包之前，你要先下个centos的容器，然后把jdk装到这个容器里面，然后打包，所以，容器的最终都是要有依赖，打包一个jdk，就是搞个系统里面装个jdk，所以你用jdk的容器其实就是一个安装了jdk的centos，我擦，就是这样
<!--more-->

***使用***

当然必须要在机器上安装docker环境了，安装方式有两种：在线安装和离线安装，我强烈建议使用在线安装，而且官方也建议在线安装，不然不好维护，不说别的，你要是离线安装，一大堆依赖让你下你就会感觉这个世界瞬间没有爱了，所以还是推荐在线安装，再说了，后面安装好之后，要是你没有私服，你还是要到hub上面去下镜像喏。这里之说centos下安装和使用，而且注意还要使用cetos7

***安装***
1. 添加yum源：
```shell
tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/$releasever/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
```
2. 安装docker-engine：yum install -y docker-engine
3. 可以配置开机启动：chkconfig docker on

装完了，你就说快不快。

***命令***

1. docker images ： 查看本地镜像
2. docker ps ： 查看docker 进程
3. docker run : 启动容器
4. docker --help : 很多东西，个人去看

***使用心得***

docker创建容器有两种方式，一种是commit，另一种是build。官方呢其实是推荐build这种方式的，就是自己编写dockerfile，我擦，这种方式问题又来了，如果编写dockerfile，所有的镜像只有通过命令来下载了，有的镜像本身就不是你想要的怎么办，你还是要修改，修改了还是只有用commit方式创建，也许这个地方我还没搞清楚，dockerfile还有其他更牛逼的用法，现在只是现在的理解。

``docker run -d --name centos cmd1 && cmd2``
然后要说下``-d``这个参数了，这个参数就是表示在后台运行，我靠，这个有点坑啊，比如，我有个centos的镜像，启动的时候加上了-d参数，但是你发现运行了一下， 这个系统就关了，这个叫什么后台运行啊。其实这里的后台表示的是在后台运行的``cmd1&&cmd2``命令，这个命令运行完之后这个系统就关了。。。。

所以为了保证你的容器在后台运行，你得保证你的cmd命令不会运行完，意思就是会一直卡在那里（不是真的卡在这里，就跟你写多线程程序一样，你同时启动10000个线程，这些线程还没运行完，但是主线程完了，所以程序只有退出了），怎么做呢，那就让它停在那儿了哦，你可以写个脚本，让他一直执行这个脚本，这样容器就不会停止了，如果是nginx你可以在启动的时候加上 ``-g "demaon off;"``参数。

还要说一点，上面的的先执行问cmd1再执行cmd2，这个没问题，但是有的时候你启动的时候，比如你手动安装的nginx-clojure，你也不配置环境变量啥的，你可以之直接到根目录下面执行``./nginx``就行了，这样没问题，但是我要说的是，如果你的cmd是绝对路径的话，这样nginx肯定是起不来的（eg:/usr/local/nginx/nginx）,那么你会这样写整个路径``docker run -d --name centos cd /usr/local/nginx && ./nginx``这样貌似看着可以，但是并没有什么卵用，你先cd到nginx目录后，执行第二个命令，他又跳出来了。。。所以我建议在打包的时候写个脚本来启动nginx：
```shell
#!/bin/sh
cd /usr/local/nginx
./nginx -g "demon off;" 
```
所以你启动的时候就成了这个样子，``docker run -d --name centos /root/nginxStart.sh``,这样是不是很方便

还有比较好用的就是-v这个参数，映射宿主机和容器磁盘的，比如``docker run -d --name -v /data/nginxlog:/usr/local/nginx/logs centos /root/nginxStart.sh`` ,这样启动后，nginx的日志就在宿主机的/data/nginxlog文件夹下了，如果你想往容器里面传文件的话，可以使用这种方式比如：``docker run -d --name -v /data:/root/data centos /root/nginxStart.sh``，这样你就可以在你的容器里面的/root/data目录下取到宿主机上的文件了。这里只是-v的两种使用方式，还有更多使用方式，还请后续关注。