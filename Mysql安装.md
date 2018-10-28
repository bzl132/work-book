##解压安装包
解压的目录就是mysql的根目录
##配置环境变量
>"%Mysql_Home%\bin

##配置my.ini文件
文件在根目录下，与bin同目录
```
[client] 
port=3306 

[mysqld] 
port=3306 
basedir="C://Program Files//Mysql"
datadir="C://Program Files//Mysql//Data"
character-set-server=utf8 
default-storage-engine=INNODB 
explicit_defaults_for_timestamp=true 

[mysql] 
no-beep 
default-character-set=utf8
```
同时创建Data文件夹，作为数据存储区，放在任何地方皆可，只要在my.ini中配置好
##初始化Mysql
初始化语句：
>mysqld --initialize-insecure --user=mysql
>mysqld -install

my.ini中配置explicit_defaults_for_timestamp=true会解决下面问题
>"TIMESTAMP with implicit DEFAULT value is deprecated.

清除根目录Data目录中的数据，否则会报下面的错误
>" --initialize specified but the data directory has files in it. Aborting.

##启动Mysql
>net start mysql

关闭mysql
>net stop mysql

## 修改密码
###方法1： 用SET PASSWORD命令
首先登录MySQL。 
格式：mysql> set password for 用户名@localhost = password('新密码'); 
例子：mysql> set password for root@localhost = password('123'); 

###方法2：用mysqladmin
格式：mysqladmin -u用户名 -p旧密码 password 新密码 
例子：mysqladmin -uroot -p123456 password 123 

###方法3：用UPDATE直接编辑user表
首先登录MySQL。 
mysql> use mysql; 
mysql> update user set password=password('123') where user='root' and host='localhost'; 
mysql> flush privileges; 

###方法4：在忘记root密码的时候，可以这样

以windows为例:
1. 关闭正在运行的MySQL服务。 
2. 打开DOS窗口，转到mysql\bin目录。 
3. 输入mysqld --skip-grant-tables 回车。--skip-grant-tables 的意思是启动MySQL服务的时候跳过权限表认证。 
4. 再开一个DOS窗口（因为刚才那个DOS窗口已经不能动了），转到mysql\bin目录。 
5. 输入mysql回车，如果成功，将出现MySQL提示符 >。 
6. 连接权限数据库： use mysql; 。 
7. 改密码：update user set password=password("123") where user="root";（别忘了最后加分号） 。 
8. 刷新权限（必须步骤）：flush privileges;　。 
9. 退出 quit。 
10. 注销系统，再进入，使用用户名root和刚才设置的新密码123登录。