1、工单管理：https://issues.sonatype.org/secure/Dashboard.jspa



## 1、创建工单

在上述的`工单管理`的地址中进行创建，如果没有账号，需要先注册一个，记住用户名密码，后边要配置到`setting.xml`中。 
Create Issue 填写内容说明： 

```
===Step 1===
Project：Community Support - Open Source Project Repository Hosting
Issue Type：New Project

===Step 2===
Summary：JAR包名称，如：marathon-client
Group Id：你懂得，不用多说，如com.cloudnil
Project URL：项目站点，如：https://github.com/CloudNil/marathon-client
SCM url：项目源码仓库，如：https://github.com/CloudNil/marathon-client.git
```

其他内容不用填写，创建Issue后需要等待一小段时间，Sonatype的工作人员审核处理，速度还是很快的，一般一个工作日以内，当Issue的Status变为RESOLVED后，就可以进行下一步操作了，否则，就等待… 

如果注册的组织id 是自己的域名 要配TXT域名解析。。。

## 2、配置Maven

在工程的pom.xml文件中，引入Sonatype官方的一个通用配置`oss-parent`，这样做的好处是很多pom.xml的发布配置不需要自己配置了：

```
    <parent>
        <groupId>org.sonatype.oss</groupId>
        <artifactId>oss-parent</artifactId>
        <version>7</version>
    </parent>
```

并增加Licenses、SCM、Developers信息：

``` 
   <licenses>
        <license>
            <name>The Apache Software License, Version 2.0</name>
            <url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
            <distribution>repo</distribution>
        </license>
    </licenses>
    <scm>
        <tag>master</tag>
        <url>git@github.com:cloudnil/marathon-client.git</url>
        <connection>scm:git:git@github.com:cloudnil/marathon-client.git</connection>
        <developerConnection>scm:git:git@github.com:cloudnil/marathon-client.git</developerConnection>
    </scm>
    <developers>
        <developer>
            <name>cloudnil</name>
            <email>cloudnil@126.com</email>
            <organization>CloudNil</organization>
        </developer>
    </developers>
```

修改maven配置文件setting.xml，在servers中增加server配置

```
  <servers>
    <server>
      <id>sonatype-nexus-snapshots</id>
      <username>Sonatype 账号</username>
      <password>Sonatype 密码</password>
    </server>
    <server>
      <id>sonatype-nexus-staging</id>
      <username>Sonatype 账号</username>
      <password>Sonatype 密码</password>
    </server>
  </servers>
```



### 3、配置gpg-key

 如果是使用的windows，可以下载gpg4win，地址：https://www.gpg4win.org/download.html，安装后在命令行中执行 gpg --gen-key生成，过程中需要填写名字、邮箱等，其他步骤可以使用默认值，不过有个叫：Passphase的参数需要记住，这个相当于是是密钥的密码，下一步发布过程中进行签名操作的时候会用到。

## 4、Deploy

这步就简单了，就是一套命令：

```
mvn clean deploy -P sonatype-oss-release -Darguments="gpg.passphrase=密钥密码"1
```

如果发布成功，就可以到构件仓库中查看了。https://oss.sonatype.org/#stagingRepositories

5、Release
进入https://oss.sonatype.org/#stagingRepositories查看发布好的构件，点击左侧的Staging Repositories，一般最后一个就是刚刚发布的jar了，此时的构件状态为open。 
打开命令行窗口，查看gpg key并上传到第三方的key验证库：
--------------------- 
```
gpg --list-keys

gpg --keyserver http://keys.gnupg.net:11371/ --send-keys 9660308C314094BC8B0510A47361DFFEF05E9909
gpg --keyserver http://pool.sks-keyservers.net:11371/ --send-keys 9660308C314094BC8B0510A47361DFFEF05E9909
gpg --keyserver http://keyserver.ubuntu.com:11371/ --send-keys 9660308C314094BC8B0510A47361DFFEF05E9909
```

回到https://oss.sonatype.org/#stagingRepositories，选中刚才发布的构件，并点击上方的close–>Confirm，在下边的Activity选项卡中查看状态，当状态变成closed后，执行Release–>Confirm，并在下边的Activity选项卡中查看状态，成功后构件自动删除，一小段时间（约1-2个小时）后即可同步到maven的中央仓库。