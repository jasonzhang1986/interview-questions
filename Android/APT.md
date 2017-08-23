[Android 编译期代码生成之 apt ](http://alighters.com/blog/2016/05/10/apt-code-generate/)

实践过程中遇到几个问题：
* 使用 AndroidStudio3.0-beta 版本，gradle 默认是 3.0.0-beta2，按照博客的说明实现，会发现在 app 的 build/generated/source/apt 目录下并没有生成 java 文件，此时我们在 Studio 的 Messages 控制台中会看到提示:

    <font color="red"><b>warning:android-apt plugin is incompatible with future version of android gradle plugin. please use 'annotationprocessor' configuration instead.</b></font>

    Google搜索之后发现，新版本的 Gradle 建议使用 annotationprocessor 工具，于是做了如下修改：
    - 去除 project 跟目录下 build.gradle 中 classpath ‘com.neenbedankt.gradle.plugins:android-apt:1.8’ 语句
    - 将 app module 下 build.gradle 中 <font color="red">apt</font> project(':compiler')  改为 <font color="red">annotationProcessor</font> project(':compiler')

* kotlin 的代码无法直接使用 annotation ，翻阅 kotlin 官网发现，需要使用 kotlin 的注解处理工具 -- [kapt](https://www.kotlincn.net/docs/reference/kapt.html)
