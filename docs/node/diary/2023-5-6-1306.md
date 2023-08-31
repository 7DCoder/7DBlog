# docker安装mysql


```sh
# 查询MySQL
docker search mysql

# 安装MySQL
docker pull mysql 

# 查看镜像
docker images

# 在opt下创建文件夹
cd /opt/
mkdir mysql_docker
cd mysql_docker/
echo $PWD

# 启动mysql容器，在var/lib/docker/containers/下查看容器
docker run --name mysqlserver -v $PWD/conf:/etc/mysql/conf.d -v $PWD/logs:/logs -v $PWD/data:/var/lib/mysql --restart always -e TZ=Asia/Shanghai -e MYSQL_ROOT_PASSWORD=123456 -d -i -p 3306:3306 mysql:latest

# 进入mysql容器，并登陆mysql
docker exec -it mysqlserver bash
mysql -uroot -p
123456

# 开启远程访问权限
use mysql;
select host,user from user;
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
flush privileges;
# 镜像里面 root用户已经有远程连接权限在里面，所以不需要去设置，只是模式不一样才导致无法连接，把root用户的密码改成 mysql_native_password 模式，即可远程连接

# 查看容器是否启动成功
docker ps
# 可以看到mysql容器起来
```