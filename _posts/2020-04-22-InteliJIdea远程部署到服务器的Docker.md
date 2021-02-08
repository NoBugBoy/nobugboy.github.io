# Idea通过Docker插件部署Java应用（看这一篇就足够了）
这里挺多内容其实在其他人的博客中都安装步骤和简单介绍（说一不说二，估计都是复制粘贴），我这里就简单说一下，主要说其中的问题和解决方案

主要的流程，步骤
 1. 安装docker

> 不多赘述了

 2. 开启docker远程端口

>  网上绝大部分介绍就是直接加上 -H tcp：0.0.0.0:2376 就完事了，确实好用，也确实有问题，分分钟成为别人矿机，就算你启用了安全组也可以直接连接，可以再另外一台装有docker的机器上尝试连接
>  docker   -H tcp://123.123.123.123:2376  ps

```java
   vi /usr/lib/systemd/system/docker.service
   //修改为下面这段
   ExecStart =/usr/bin/dockerd -H tcp://0.0.0.0:2376 -H unix:///var/run/docker.sock
```
3. 打开idea安装docker插件 

> 打开idea -> file -> settings -> Plugins -> 搜索docker -> 安装即可

4. 配置插件

> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200422085654623.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)

5. 配置pom.xml

> 添加docker-maven-plugin 这个很简单网上很有多可以搜素一下（复制粘贴的）
```xml
<plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>1.0.0</version>

                <!--将插件绑定在某个phase执行-->
                <executions>
                    <execution>
                        <id>build-image</id>
                        <!--将插件绑定在package这个phase上。也就是说，
                        用户只需执行mvn package ，就会自动执行mvn docker:build-->
                        <phase>package</phase>
                        <goals>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>

                <configuration>
                    <!--指定生成的镜像名,这里是我们的项目名-->
                    <imageName>${project.artifactId}</imageName>
                    <!--指定标签 这里指定的是镜像的版本，我们默认版本是latest-->
                    <imageTags>
                        <imageTag>latest</imageTag>
                    </imageTags>
                    <!-- 指定我们项目中Dockerfile文件的路径-->
                    <dockerDirectory>${project.basedir}/src/main/resources</dockerDirectory>

                    <!--指定远程docker 地址-->
                    <dockerHost>http://123.123.123.123:2376</dockerHost>
                    <!-- 这里是复制 jar 包到 docker 容器指定目录配置 -->
                    <resources>
                        <resource>
                            <targetPath>/</targetPath>
                            <!--jar包所在的路径  此处配置的即对应项目中target目录-->
                            <directory>${project.build.directory}</directory>
                            <!-- 需要包含的 jar包 ，这里对应的是 Dockerfile中添加的文件名　-->
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                </configuration>
            </plugin>
```
6. 将Dockerfile文件放入项目根路径下（**和pom.xml同一级**）即可，当使用mvn package（**也可以改为其他命令**）时就会自动打包，并且生成镜像（**不会自动启动容器，需要手动**）

### 以上就是普通版本的配置，会有安全问题

> 起初我以为是安全组未生效的问题，后来以为是docker中redis的问题，分别进行了打补丁之后发现依然没有解决问题，最后能出问题的只有docker本身了，因为在未启用远程端口时并无此类问题，于是使用了远程加密连接方案

 1. 首先使用openssl生成证书

> 网上有许多教程这里就不多讲了，只需要注意如果是IP访问，下面要写IP：否则写DNS
> echo subjectAltName = IP:123.123.123.123,IP:0.0.0.0 >> extfile.cnf
> 只挂一个官方的安全连接方案地址：[超级无敌传送门](https://docs.docker.com/engine/security/https/)

 2. 修改上面第二步的配置

> ExecStart=/usr/bin/dockerd --tlsverify --tlscacert=/etc/docker/ca.pem --tlscert=/etc/docker/server-cert.pem --tlskey=/etc/docker/server-key.
> pem -H tcp://0.0.0.0:2376 -H unix:///var/run/docker.sock
> 注意改为自己刚刚生成证书的目录，修改好记得重启
3. 重新打开idea修改上面第4步的docker插件配置
>  改为Https，并且将服务器上生成的证书拿到本地（只需要ca.pem,cert.pem,key.pem）放入一个文件夹中，然后按照图片配置好文件夹路径![在这里插入图片描述](https://img-blog.csdnimg.cn/20200422091512992.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
4. 最后还需要两个操作，第一步将上面存放证书的文件夹三个文件拷贝到项目路径中（**反正能找到就行**），第二步加一个dockerCertPath标签配置刚才放三个文件的位置
  ```xml
    <dockerHost>https://123.123.123.123:2376</dockerHost>
    <dockerCertPath>${project.basedir}/src/main/resources</dockerCertPath>
  ```
5. 到这里才算完全的配置完成整套安装流程（撒花）
### 下面就可以开始愉快的使用环节
还不能撒花，在build image时候如果出现问题请检查上述步骤是否有误，检查Dockerfile是否有问题等,如果连接有问题检查2376端口是否被放开

我的idea是2019.3版本，打开services面板，配置一下各种启动属性（其实就是配置docker run xxxxx）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200422093327254.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020042209325826.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200422093401959.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
终于可以撒花了
### 总结一下
可以在idea面板内对docker进行一些操作，看到所有镜像容器的配置，可以直接修改，能实时看到日志，构建镜像速度其实还是有点慢，运行容器的速度是很快的，如果项目和其他中间件等是docker部署可以考虑使用这种简易的方案，还是挺方便的