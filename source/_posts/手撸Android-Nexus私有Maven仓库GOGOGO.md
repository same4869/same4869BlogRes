---
title: 手撸Android-Nexus私有Maven仓库GOGOGO
date: 2019-3-22
tags: [Android]
---
> 我们在做Android SDK的时候，总是以jar或者aar的方式让业务方依赖，在这条流水线建立起来的时候，手动去把依赖包丢给业务方显然不是正确的打开方式，而作为公司层面，上传到jcenter又会顾忌到安全的敏感问题，于是就有了私有maven仓库的出现，能够很好的解决以上问题，本文描述就是一个建立私有maven的过程，工具准备好了，总会有用到的时候。

<!-- more -->

#### 1.远程仓库

![Alt text](/img/015/20190322-01.jpg)

一图胜千言，不过还是描述下，一般来说仓库分为本地的和远程的。

本地的没啥好说的，本地硬盘的一个路径，或者放在libs文件夹下来手动集成，某种程度上都算。

远程的又分为中央仓库和私库，中央仓库比较出名的是 `JCenter` 和 `Maven Centra`，全世界开源的库都可以往上面传，谁也都可以去依赖到。虽然开源是提倡的，但并适用于任何场景，比如公司的核心依赖库，在这种情况下，在公司的内网才能访问的服务器上部署Nexus私库的场景就应运而生了。

#### 2.准备搭建

首先是需要JDK1.8以上的JAVA环境，这个自备一下。

Nexus的2.X和3.X的版本感觉差了不少，本文使用的是最新的3.X的，下载官方网站是`https://www.sonatype.com/download-oss-sonatype`，可以根据自己的平台选择，这里使用的是MACOS环境，远程服务器测试是使用了阿里云的centos7.x。

![Alt text](/img/015/20190322-02.png)

下载后解压大概目录如下图所示

![Alt text](/img/015/20190322-03.jpg)

直接运行bin目录下面的nexus就行了，linux和macos的小伙伴直接运行

`./nexus run`就行了（貌似也可以用`./nexus start`和`./nexus stop`配套指令来开启和关闭）

#### 3.初试牛刀

顺利运行起来了的话就可以看效果了，默认的端口是8081，当然可以在对应的配置文件里面去修改默认端口号，如果是本地起的服务，可以直接在浏览器里输入`http://localhost:8081/`，效果如下

![Alt text](/img/015/20190322-04.jpg)

> 这个服务运行起来大概需要几百兆的空闲内存，不然会跑不起来（第一次说需要400多M，第二次800多M）。

默认是没登录的，Nexus 提供了一个完全访问权限的管理用户。 用户名是 `admin`，密码是`admin123`。

左边切换到browser选项可以到仓库界面，如图

![Alt text](/img/015/20190322-05.jpg)

注意 Type 列，展示了 Nexus 的仓库分类：

- proxy（远程代理仓库）
这种类型的仓库，可以设置一个远程仓库的链接。当用户向 proxy 类型仓库请求下载一个依赖构件时，就会先在自己的库里查找，如果找不到的话，就会从设置的远程仓库下载到自己的库里，然后返回给用户，相当于起到一个中转的作用。

	例如 maven-central 用来存储从 Maven 中央仓库下载过的构件。

- group （聚合仓库）
在 Maven 里没有这个概念，是 Nexus 特有的。目的是将多个仓库聚合，对用户暴露统一的地址，这样用户就不需要配置多个地址，只要统一配置 group 的地址就可以了。group 仓库的聚合成员可以在仓库设置中添加和移除。

	例如 maven-public 是一个 group 类型的仓库，通过引用这个地址，可以访问组内成员仓库的所有构件。
	
	![Alt text](/img/015/20190322-06.jpg)
	
- hosted（宿主仓库）
我们自己的构件，上传的就是这样的仓库。目前 maven-releases 和 maven-snapshots 是 hosted 类型的仓库。我们可以上传到这两个仓库，也可以自己创建 hosted 仓库。

#### 4.Android工程上传到Nexus私库

现在本地的私库已经跑起来了，接下来开始配置Android工程,看看要怎么上传到私库了。

首先我们来创建一个Android工程，是需要打包成aar上传到私服的，所以这个工程必须是`library`，也就是在`build.gradle`里配置成`apply plugin: 'com.android.library'`。

然后需要在这个library工程的模块内的gradle中配置一些task，为了方便可以单独抽一个文件出来，叫做`nexus.gradle`,放在根目录，内容如下：

```
apply plugin: 'maven'
version=LIB_VERSION
def nexusRepositoryUrl = NEXUS_RELEASES
if (!LIB_IS_RELEASE.toBoolean()){
    version = "${version}-SNAPSHOT"
    nexusRepositoryUrl = NEXUS_SNAPSHOTS
}
task sourcesJar(type: Jar) {
    classifier = 'sources'
    from android.sourceSets.main.java.sourceFiles
}
task javadoc(type: Javadoc) {
    source = android.sourceSets.main.java.sourceFiles
    classpath += project.files(android.getBootClasspath().join(File.pathSeparator))
    classpath += configurations.compile
    options {
        failOnError false
        encoding "UTF-8"
        charSet 'UTF-8'
        author true
        version true
        links "http://docs.oracle.com/javase/7/docs/api"
    }
}
task javadocJar(type: Jar, dependsOn: javadoc) {
    classifier = 'javadoc'
    from javadoc.destinationDir
}
artifacts {
    archives javadocJar
    archives sourcesJar
}
uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: "$nexusRepositoryUrl") {
                authentication(userName: NEXUS_USERNAME, password: NEXUS_PASSWORD)
            }
            pom.project {
                name LIB_ARTIFACT
                groupId LIB_GROUP
                artifactId LIB_ARTIFACT
                version version
                packaging 'aar'
                description LIB_DES
            }
        }
    }
}
```

然后在moudle中的gradle添加`apply from: '../nexus.gradle'`。

可以看到，在`nexus.gradle`这个脚本中有一些常量，这些常量需要写在
`gradle.properties`中，相当于配置的参数。

```
NEXUS_SNAPSHOTS=http://yourhost:8081/repository/maven-snapshots/
NEXUS_RELEASES=http://yourhost:8081/repository/maven-releases/

NEXUS_USERNAME=admin
NEXUS_PASSWORD=admin123

LIB_DES=hj web mch5web

LIB_GROUP=com.xun.mch5web
LIB_ARTIFACT=browser
LIB_VERSION=1.0.5
LIB_IS_RELEASE=false //如果是true就是release版本，如果是false就是snapshot版本，release版本不可版本覆盖，需管理员手动删除
```

看变量名基本也能猜出每个变量的含义，这里也不在赘述了。

准备工作就绪，可以安心专注于Android SDK的开发了，开发完了就需要上传，直接在项目目录控制台运行`./gradlew clean build upload`（其实upload就行，编译后上传）。

还有种方法，Android Studio 中打开右侧的 Gradle 侧边栏，打开项目名,可以看到`uploadArchives`,这就是刚才创建的上传 Task,点击即可完成上传。

以上，成功之后可以在Nexus仓库中看到上传的相关文件。

![Alt text](/img/015/20190322-07.jpg)

#### 5.Android工程依赖Nexus相关包

上传已经OK，如果有一个Android工程（Application）想依赖刚刚上传的library，应该这么玩。

首先在`rootProject`的`build.gradle`中引用私库地址，我引用的是 `maven-public`聚合仓库的地址：

```
maven { url 'http://yourhost:8081/repository/maven-public/' }
```

引用该三方库的目标`Module`的`build.gradle`中添加此库的依赖：

`compile 'com.xun.mch5web:browser:1.0.5-SNAPSHOT'`

snapshot版本是可以版本覆盖的，有时候依赖会有缓存，可以尝试使用以下命令清除缓存

`./gradlew build --refresh-dependencies `

#### 6.将nexus部署到远程服务器和迁移相关

以上整个流程算是通了，不过在实际运用中，nexus肯定是需要部署在公司内网的物理或者虚拟机上的，下面用阿里云的云主机做一个模拟（系统是centos7.X）。

JDK环境确认没问题之后，使用`wget 下载url`命令把nexus的压缩包

![Alt text](/img/015/20190322-08.jpg)

上图下载了两个版本，使用的是3.15那个。

然后根据前文按部就班，运行起来之后就是`远程主机ip+8081（默认）`的方式被外网访问（nexus运行之后可以`ps aux | grep "nexus"`看看进程是否存在，阿里云的主机需要在阿里云后台去开启对应的端口，这个非常坑，千万不要忘了）。

![Alt text](/img/015/20190322-09.jpg)

至于用法也是一样的，就不多说了。

再来简单说一下迁移的事情，前面我们可以看到，在nexus解压包中会解压出一个名叫`sonatype-work`的文件夹，平时备份好这个文件夹的所有文件就OK了，还原的时候可以直接进行替换。nexus2.X的方式可能有些不一样，暂时还没研究过。

#### 7.总结

内容以上，公司如果又横向团队，这个也算是必备的技术了，高效率总是一点一滴累积出来的，技术没有高尚与低贱，能提高生产力的，就是好的。


