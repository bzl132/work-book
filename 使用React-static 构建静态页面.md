## 使用React-static 构建静态页面

React Static 是一个 React 的渐进式静态网站生成器。它也是一个服务端渲染 React 应用的简约框架，旨在构建一个满足 SEO，网站性能和用户/开发人员使用体验的标准，帮助每个人无痛地构建下一代、高性能的网站。 

代码库地址:https://github.com/nozzle/react-static

###全局安装

```
$ npm install -g react-static
```

###使用脚手架

创建项目

```
react-static create
```

构建项目生成dist包

```
react-static build
```



### 服务器部署dist

服务器安装nodejs

```
wget http://nodejs.org/dist/v11.3.0/node-v11.3.0.tar.gz

npm install -g n #安装nodejs版本更新工具
n latest #更新nodejs 为最新版本
tar xzvf node
./configure
make
make install
```

直接修改nginx配置即可,如下

```
        root   /home/react/demo;

        access_log  logs/nginx.access.log  main;

        location / {
           index index.html;
        }
```





**未用到pm2**

安装pm2 服务端管理工具:

```
npm install -g pm2
```

主要是后面的两个警告：

```
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: fsevents@1.1.3 (node_modules/pm2/node_modules/fsevents):

npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for fsevents@1.1.3: wanted {"os":"darwin","arch":"any"} (current: {"os":"linux","arch":"x64"})

```


通常npm安装的时候有警告可以直接忽略。这里其实也是。但是执行pm2命令就一直提示不存在pm2这个时候就需要把pm2配置全局变量了。

```
ln -s /usr/local/src/node-v8.9.0-linux-x64/bin/pm2  /usr/local/bin/pm2
```