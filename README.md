# fastdfsOnDocker
该文档是构建fastdfs的tracker节点以及storage节点镜像的相关说明。

fastdfs是一个开源的分布式文件系统，其主要分为tracker节点和storage节点，本镜像部署了storage节点，并在镜像内构建了一个nginx服务器，支持http服务。在docker的环境中，tracker节点部署在了与storage节点相同的服务器上。在构建服务前，需要更改其具体的配置文件。在运行之前，请先下载如下文件：

fastdfs-5.05.tar.gz

libfastcommon-master.zip

fastdfs-nginx-module-master.zip

nginx-1.6.3.tar.gz

ngx_cache_purge-2.3.tar.gz

其中前三个压缩包可以在fastdfs的github上下载到，nginx能去官网下载，最后一个可以直接百度，注意下载最新的版本即2.3，否则会构建失败，然后在Dockerfile所在的目录下构建一个文件夹为file，将这些压缩包放置在file中。

下载完所有文件后，就可以构建镜像或者直接从dockerhub上pull镜像了，dockerhub上的镜像名称为fastdfs_tracker:complete以及fastdfs_storage:complete。也推荐从alauda.cn来获取镜像，因为速度很快，在alauda.cn的镜像名称为index.alauda.cn/lionheart/fastdfs_storage:complete以及index.alauda.cn/lionheart/fastdfs_tracker:latest。如果直接pull下来镜像，那么不需要进行过多的配置，只需要修改tracker，storage节点的ip地址即可。如果是以dockerfile的方式构建镜像，那么你需要参考dockerfile中的配置文件已经说明文件来进行相应的配置。我所构建的镜像是部署在一台服务器上的tracker以及storage节点，如果你需要额外的配置，即配置在多台服务器上的storage节点，那么你可能需要安装pipework以让docker支持容器间的网络通信，具体的配置请参照该文档 http://wenku.baidu.com/link?url=7p3i2ev79UnCGAsbYX9iYO78PN8IAKpZdvU-VOxZRGCEqgyR8Nrvr129Dg8yTjGWAZNR945gLwf1Kv9DvJmaAMSCyxoZuop8d3qvYl1OdHy

如果你需要从dockerfile构建自己的镜像，那么请详细阅读dockerfile文件夹以及script文件夹中的说明文档，按照说明文档的指示来进行你的构建工作。

如下是每条语句的具体解释，你可以根据自己的要求来更改这些运行语句

先运行语句

docker run -d --name trackerconfig tracker

(trackerconfig为容器名，可为任意名字，用完后会删掉,tracker为tag后的镜像名称，也可以为其他的名称)

docker run -d --name storageconfig storage

(同上面的容器一样，也是用后会删掉)

创建 /home/fastdfs文件夹

mkdir -p /home/fastdfs/nginx_conf

mkdir -p /home/fastdfs/client

从容器中拷贝fastdfs的配置文件目录到宿主机，并将该目录重命名为fdfs_conf

docker cp storageconfig:/home/fastdfs/fastdfs-5.05/conf /home/fastdfs

mv /home/fastdfs/conf /home/fastdfs/fdfs_conf

从容器中拷贝tracker节点的nginx配置文件目录到宿主机，并将该目录重命名为tracker

docker cp trackerconfig:/home/fastdfs/fastdfs-5.05/conf /home/fastdfs/nginx_conf

mv /home/fastdfs/nginx_conf/conf /home/fastdfs/nginx_conf/tracker

从容器中拷贝storage节点的nginx配置文件目录到宿主机，并将该目录重命名为storage

docker cp storageconfig:/home/fastdfs/fastdfs-5.05/conf /home/fastdfs/nginx_conf

mv /home/fastdfs/nginx_conf/conf /home/fastdfs/nginx_conf/storage

从容器中拷贝fastdfs的配置文件目录到宿主机，并将该目录重命名为etc_conf

docker cp storageconfig:/home/fastdfs/fastdfs-5.05/conf /home/fastdfs

mv /home/fastdfs/conf /home/fastdfs/etc_conf

从容器中拷贝fastdfs的存储文件目录到宿主机

docker cp storageconfig:/home/fastdfs_file /home/fastdfs

接下来修改配置文件

进入目录/home/fastdfs/fdfs_conf打开storage.conf文件，修改如下

tracker_server=ipaddress:22122

(ipaddress为具体的ip地址，为你所部署的该节点的ip地址，如192.168.0.1，该ip地址不能为localhost或者127.0.0.1，如果将tracker和storage节点部署在一台服务器上，那么就用其对外公开的ip地址来代替)

进入目录/home/fastdfs/fdfs_conf打开client.conf文件，修改如下

tracker_server=ipaddress:22122

运行语句如下所示：

tracker节点：

docker run -it --name tracker --net=host -v /home/fastdfs/fdfs_conf:/home/fastdfs/fastdfs-5.05/conf/ -v /home/fastdfs/nginx_conf/tracker:/usr/local/nginx/conf tracker

此处会进入docker容器，需要输入exit命令，下面的节点也是如此，不再赘述

docker start tracker

docker exec tracker fdfs_trackerd /home/fastdfs/fastdfs-5.05/conf/tracker.conf

docker exec tracker /usr/local/nginx/sbin/nginx

storage节点:

docker run -it --name storage --net=host -v /home/fastdfs/fdfs_conf:/home/fastdfs/fastdfs-5.05/conf/ -v /home/fastdfs/nginx_conf/storage:/usr/local/nginx/conf -v /home/fastdfs/etc_conf:/etc/fdfs storage

docker start storage

docker exec storage fdfs_storaged /home/fastdfs/fastdfs-5.05/conf/storage.conf

docker exec storage /usr/local/nginx/sbin/nginx

配置完成后，我们可以进行测试，fastdfs提供了一个测试用的client工具，所以可以创建一个client容器，进行测试。测试前要在/home/client文件夹下面放一个图片，起名为test.jpg。

docker run -it --name client --net=host -v /home/fastdfs/client:/home/client storage

docker start client

docker exec client fdfs_test /home/fastdfs/fastdfs-5.05/conf/client.conf upload /home/client/test.jpg

然后会返回一个url值，如下所示

http://192.168.1.107/group1/M00/00/00/wKgBa1Vd7AyABRFaAAbVsieNsu4588_big.jpg

最后在浏览器输入url值，如果是部署在服务器上，需要加上服务器tracker节点的端口号，本docker镜像默认的tracker节点的端口号为80,storage节点的端口号为8888，如果是本机部署，则直接用localhost加端口号,如果能够显示出来图像，则证明配置成功。
