###在kubernetes中搭建LNMP环境，并安装Discuz

说明： 本文档为课堂教学辅助项目，不做太多解释，如果有疑问可以加本人微信81677956讨论。做本实验，需要已经搭建好kubernetes集群和harbor服务。

首先克隆本项目：git clone  https://git.coding.net/aminglinux/k8s_discuz.git

####下载镜像
```
docker pull mysql:5.7
docker pull richarvey/nginx-php-fpm
```

####用dockerfile重建nginx-php-fpm镜像
```
cd k8s_discuz/dz_web_dockerfile/
docker build -t nginx-php .

```

####将镜像push到harbor
```
##登录harbor，并push新的镜像
docker login harbor.yuankeedu.com  //输入正确的用户名和密码
docker tag nginx-php  harbor.yuankeedu.com/aminglinux/nginx-php
docker push harbor.yuankeedu.com/aminglinux/nginx-php
docker tag mysql:5.7 harbor.yuankeedu.com/aminglinux/mysql:5.7
docker push harbor.yuankeedu.com/aminglinux/mysql:5.7
```

####搭建NFS

假设kubernetes集群网段为172.7.5.0/24，本机IP为172.7.5.13
* 安装包
```
yum install nfs-utils
```
* 编辑配置文件
```
vim /etc/exportfs  //内容如下
/data/k8s/ 172.7.5.0/24(sync,rw,no_root_squash)
```
* 启动服务
```
systemctl start nfs
systemctl enable nfs
```
* 创建目录
```
mkdir -p  /data/k8s/discuz/{db,web}
```

####搭建MySQL服务
* 创建secret (设定mysql的root密码)
```
kubectl create secret generic mysql-pass --from-literal=password=DzPasswd1
```
* 创建pv
```
cd ../../k8s_discuz/mysql
kubectl create -f mysql-pv.yaml
```
* 创建pvc
```
kubectl create -f mysql-pvc.yaml
```
* 创建deployment
```
kubectl create -f mysql-dp.yaml 
```
* 创建service
```
kubectl create -f mysql-svc.yaml
```

####搭建Nginx+php-fpm服务
* 搭建pv
```
cd ../../k8s_discuz/nginx_php
kubectl create -f web-pv.yaml
```
* 创建pvc
```
kubectl create -f web-pvc.yaml
```
* 创建deployment
```
kubectl create -f web-dp.yaml 
```
* 创建service
```
kubectl create -f web-svc.yaml
```
####安装Discuz
* 下载dz代码 (到NFS服务器上)
```
cd /tmp/
git clone https://gitee.com/ComsenzDiscuz/DiscuzX.git
cd /data/k8s/discuz/web/
mv /tmp/DiscuzX/upload/* .
chown -R 100 data uc_server/data/ uc_client/data/ config/
```
* 设置MySQL普通用户
```
kubectl get svc dz-mysql //查看service的cluster-ip，我的是10.68.122.120
mysql -uroot -h10.68.122.120 -pDzPasswd1  //这里的密码是在上面步骤中设置的那个密码
> create database dz;
> grant all on dz.* to 'dz'@'%' identified by 'dz-passwd-123';
```
* 设置Nginx代理
```
注意：目前nginx服务是运行在kubernetes集群里，node节点以及master节点上是可以通过cluster-ip访问到，但是外部的客户端就不能访问了。
      所以，可以在任意一台node或者master上建一个nginx反向代理即可访问到集群内的nginx。
kubectl get svc dz-web //查看cluster-ip，我的ip是10.68.190.99
nginx代理配置文件内容如下：
server {
            listen 80;
            server_name dz.yuankeedu.com;

            location / {
                proxy_pass      http://10.68.190.99:80;
                proxy_set_header Host   $host;
                proxy_set_header X-Real-IP      $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
}
```

* 安装Discuz
```
做完Nginx代理，就可以通过node的IP来访问discuz了。
```