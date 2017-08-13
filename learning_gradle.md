# Gradle 

---
Gradle是一种构建工具，使用Groovy语言来描述构建配置，可读性和可维护性强，可以用来构建java、Android、iOS、C/C++等工程。
Groovy是一种基于JVM的敏捷开发语言，结合了python、Ruby、Smalltalk的许多强大特性，能够与java代码很好结合，与java互相操作非常容易。

[TOC]

-----
## Gradle 环境变量
> export GRADLE_HOME=/your_gradle_directory
> export PATH=\$PATH:\$GRADLE_HOME/bin

##Gradle Task

Gradle的任务都是以task驱动，每个task以闭包closure的形式进行配置，一个task也是一个action，是可以被执行的action。每个closure也是一个对象，可以当作参数传递。

###Declare Task

*vim build.gradle*  

```
task hello  
```

这就声明了一个名为hello的task

###Task Action
声明了一task，但task需要做事情，就是action，下面是build.gradle文件描述task action的形式。

```
task hello

hello << {
	print “hello"
}

hello << {
	println " world!"
}
```
执行gradle构建的结果:  
> \# gradle -b build.gradle hello  
> hello world!  


-b参数是表示指定gradle的构建文件，不加情况下默认当前目录下的build.gradle  
hello是构建要执行的任务名称

**gralde文件里面，“<<“ 符号表示是同一个task继续执行的部分，这样一个task可以分开多个地方来写。跟shell命令行append内容到一个文件下的含义类似。**


###Task Configuration
当gralde开始构建时，会经历initialization、configuration、execution的过程

```flow
st=>start: initialization
op1=>operation: configuration
op2=>operation: execution

st->op1->op2
```
* initialization是gradle决定哪些project要参与本次构建的过程   
* configuration是gradle将这些task被汇编为内部object model的时候的过程，通常叫做DAG(*derected acyclic graph* 无循环的有向图)。
* execution是gradle按照task的依赖关系来执行task的过程

上面一个task带"<<"符号的closure是属于一个execution，我们可以增加一个configuration
```
task hello

hello << {
	print 'hello'
}

hello << {
	println ' world!'
}

hello {
    println "start configure hello task"
}
```

> \# gradle hello  
> start configure hello task  
> :hello  
> hello world!  

可以看到，“start configure hello task"虽然加在构建文件的后面，但却先执行了，因为这是在gradle构建的configuration阶段执行的，后面的closure是execution阶段执行的。

###Task Object
所有的Task都是一个Object，都是一个closure，像其他语言一样，每一个object都接受一个DefaultTask，类似继承一样，每一个object都具有DefaultTask具有的方法Method和属性Propoerties。

####DefaultTask的方法  
* **dependOn(task)**  
dependOn对单个依赖的不同描述方式
```
// Declare that world depends on hello
// Preserves any previously defined dependencies as well
task loadTestData {
    dependsOn createSchema
}
// An alternate way to express the same dependency
task loadTestData {
    dependsOn << createSchema
}
// Do the same using single quotes (which are usually optional)
task loadTestData {
    dependsOn 'createSchema'
}
// Explicitly call the method on the task object
task loadTestData
loadTestData.dependsOn createSchema
// A shortcut for declaring dependencies
task loadTestData(dependsOn: createSchema)
```
dependOn对多个依赖的不同描诉方式
```
// Declare dependencies one at a time
task loadTestData {
    dependsOn << compileTestClasses
    dependsOn << createSchema
}
// Pass dependencies as a variable-length list
task world {
    dependsOn compileTestClasses, createSchema
}
Tasks Are Objects | 17// Explicitly call the method on the task object
task world
    world.dependsOn compileTestClasses, createSchema
// A shortcut for dependencies only
// Note the Groovy list syntax
task world(dependsOn: [ compileTestClasses, createSchema ])
```
* **doFirst(closure)**
在每个task execution阶段开始之前执行的动作，可以设置多个doFirst串联执行
```
// Initial task definition (maybe not easily editable)
task setupDatabaseTests << {
    println 'load test data'
}
setupDatabaseTests.doFirst {
    println 'create database schema'
}
setupDatabaseTests.doFirst {
    println 'drop database schema'
}
```
执行结果
> \# gradle world  
> :setupDatabaseTests  
> drop database schema  
> create database schema  
> load test data  

也可以在configuration的closure里面加
```
// Initial task definition (maybe not easily editable)
task setupDatabaseTests << {
    println 'load test data'
}
// Our changes to the task (in a place we can edit them)
setupDatabaseTests {
    doFirst {
        println 'create database schema'
    }
    doFirst {
        println 'drop database schema'
    }
}
```
* **doLast(closure)**
 在每个task execution阶段结束之后执行的动作，可以设置多个doLast串联执行
```
task setupDatabaseTests << {
    println 'create database schema'
}
setupDatabaseTests.doLast {
    println 'load test data'
}
setupDatabaseTests.doLast {
    println 'update version table'
}
```
* **onlyif(closure)**
当onlyif里面的closure返回真的时候，这个task才执行，否则这个task跳过

####DefaultTask的properties
* **didWork**  ——boolean属性表示task是否已经完成
* **enabled**  ——boolean属性表示task是否会被执行
* **path** ——string属性表示task到层次路径，比如:hello:helloworld
* **loger** ——Gradle内部的logger对象引用
* **logging** —The logging property gives us access to the log leve
* **description** ——Task的一些描述信息
* **temporaryDir** ——指向一个属于这个构建文件的临时目录的File 对象  

####其他任意的动态产生的properties
跟其他语言一样，closure里面可以自己创建一个property，并且可以被引用。一个task就好像一个hash map，里面可以放任意的key - value形式的function和property，例如：
```
task copyFiles {
    // Find files from wherever, copy them
    // (then hardcode a list of files for illustration)
    fileManifest = [ 'data.csv', 'config.json' ]
}
task createArtifact(dependsOn: copyFiles) << {
    println "FILES IN MANIFEST: ${copyFiles.fileManifest}"
}
```

###自定义的Task类型
task类型可以自定义，有些是库已经定义好的，例如Copy，Jar，JavaExec，用于执行特定的任务，还可以通过extend DefaultTask的方式自定义一些执行特定任务的Task
* Copy——拷贝文件
```
task copyFiles(type: Copy) {
    from 'resources'
    into 'target'
    include '**/*.xml', '**/*.txt', '**/*.properties'
}
```
* Jar——把class文件打包成Jar包
```
apply plugin: 'java'
task customJar(type: Jar) {
    manifest {
        attributes firstKey: 'firstValue', secondKey: 'secondValue'
    }
    archiveName = 'hello.jar'
    destinationDir = file("${buildDir}/jars")
    from sourceSets.main.classes

```
* JavaExec——执行class文件里面的main函数，就是执行java代码
```
apply plugin: 'java'
repositories {
    mavenCentral()
}
dependencies {
    runtime 'commons-codec:commons-codec:1.5'
}
task encode(type: JavaExec, dependsOn: classes) {
    main = 'org.gradle.example.commandline.MetaphoneEncoder'
    args = "The rain in Spain falls mainly in the plain".split().toList()
    classpath sourceSets.main.classesDir
    classpath configurations.runtime
}
```
* custom types——自定义某种类型的task，通过@TaskAction的annotaion声明来设置task的execution阶段需要执行的action
```
task createDatabase(type: MySqlTask) {
    sql = 'CREATE DATABASE IF NOT EXISTS example'
}
task createUser(type: MySqlTask, dependsOn: createDatabase) {
    sql = "GRANT ALL PRIVILEGES ON example.*
        TO exampleuser@localhost IDENTIFIED BY 'passw0rd'"
}
task createTable(type: MySqlTask, dependsOn: createUser) {
    username = 'exampleuser'
    password = 'passw0rd'
    database = 'example'
    sql = 'CREATE TABLE IF NOT EXISTS users
        (id BIGINT PRIMARY KEY, username VARCHAR(100))'
}
class MySqlTask extneds DefaultTask {
    def hostname = 'localhost'
    def port = 3306
    def sql
    def database
    def username = 'root'
    def password = 'password'

    @TaskAction
    def runQuery() {
        def cmd
        if(database) {
            cmd = "mysql -u ${username} -p${password} -h ${hostname}
                -P ${port} ${database} -e "
        }
        else {
            cmd = "mysql -u ${username} -p${password} -h ${hostname} -P ${port} -e "
        }
        project.exec {
            commandLine = cmd.split().toList() + sql
        
    }
}
```
##加速Gradle编译
It is recommand the enable gradle daemon to speed up the compling task. **But not recommand for the continue integration environment.**

To enable **gradle daemon**, edit gradle.properties:
> \# cd ~/.gradle  
> \# vim gradle.properties

add the following line:
> org.gradle.daemon=true

支持并发编译，同样在gradle.properties文件增加：
> org.gradle.parallel=true

另外在命令行可以指定并发编译执行的线程数：
> gradle build --parallel --parallel-threads=N

##Gradle Wrapper
Gradle Wrapper是一个方便的工具，可以使别人不需要预先装好gradle，就可以开始进行构建。Gradle Wrapper会自动判断是否已经下载安装好需要的gradle版本，如果没有则会进行下载安装，再根据build.gradle的配置进行构建，一条龙服务。

生成Gradle Wrapper有方便的方法，在build.gradle里面增加一个task
```
task wrapper(type: Wrapper) {
	gradleVersion = '2.4'
}
```
然后执行这个task
> \# gradle wrapper

就会生成Gradle Wrapper所需要的各种文件：  
```
simple/
  gradlew
  gradlew.bat
  gradle/wrapper/
    gradle-wrapper.jar
    gradle-wrapper.properties
```

执行gradlew(unix)或者gradlew.bat(windows)，就可以进行一条龙服务的自动化构建。

-----------
##Android  Gradle
Android studio的Android Plugin for Gradle，是gradle的一个插件，可以使得从以下几个方面配置Android的构建：
* Build variants——构建的变量，能够对通一个module，根据不同的product和build configuration，生成多个不同的APK。具体例子看下面的product flavors和buildType介绍。
* Dependences——依赖关系。
* Manifest entries——可以修改manifest文件里面某个节点的值。
* Signing——APK的签名。
* ProGuard——可以控制不同的ProGurad参数。
* Testing——可以控制一个测试项目。

详细的Android Plugin for Gradle的官方文档在这里：
http://tools.android.com/tech-docs/new-build-system
其中plugin涉及到的对象以及属性方法的DSL Reference：
http://google.github.io/android-gradle-dsl
中文翻译：
http://avatarqing.github.io/Gradle-Plugin-User-Guide-Chinese-Verision/index.html
美团gradle实践：
http://tech.meituan.com/gradle-practice.html


### Android Studio Project
Android Studio Project 是按照符合Android的gradle plugin的build功能，组织的project形式。可以包含多个module，每一个module是一个可以单独编译/调试/测试的单位，都可以包含一个build.gradle文件，module包含以下几种类型：
* *Android application modules* —— Android应用程序的工程，包含代码，可以生成APK文件。
* *Android library modules* —— 可复用的Android特有的代码和文件，构建系统会编译出来AAR(Android ARchive)文件。
* *App Engine modules*  —— 包含了App Engine的代码和资源。
* *Java library modules* ——  包含了JAR包，通用的java库。

按照层次，project最顶层有一个project层次的build.gradle文件，每个module又分别有每个module的build.gradle文件。

#### Project build.gradle文件
缺省情况下，这里包含了整个project适用的配置，有buildscript任务和依赖，主要功能还是标注了远程的依赖关系，一般不会修改到这里，也不建议修改到这里，把修改都放到每个module的build.gradle里面。Project的build.gradle例子：
```
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:1.0.1'
        // NOTE: Do not place your application dependencies here: they belong
        // in the individual module build.gradle files
    }
}

allprojects {
   repositories {
       jcenter()
   }
}
```
#### Module build.gradle文件
* android settings
	* compileSdkVersion
	* buildToolsVersion
* defaultConfig and productFlavors 
	* manifest文件的属性，例如applicationId，miniSdkVersion，targetSdkVersion等信息可以指定
	* product flavors用来比较方便得在一个project上构建出来不同版本的产品，比如不同渠道的产品包，收费和免费的产品包，等等，每一个product flavor，能生成一个新的APK。
* buildTypes
	* build properties such as debuggable, ProGuard enabling, debug signing, version name suffix and test information
	* 每一个buildType，也能生成一个新的APK，例如，buildType指定了debug、和release两种buildType。

#### Build variants 构建的变量以及渠道号打包的实现
上面对build types和product flavors的简短介绍可以知道，它们都分别可以配置生成新的APK。根据它们所控制的含义不同，实际上每一个(product flavor, build type)对，可以生成一个新的APK，这称为build variants，例如：
* (应用宝product,  debug)
* (应用宝product,  release)
* (小米应用市场product,  debug)
* (小米应用市场product,  release)
* (免费product,  debug)
* (免费product,  release)
* (收费product,  debug)
* (收费product,  release)

实际应用上，product flavors可以和[manifest-merger里面的Placeholder Support](http://tools.android.com/tech-docs/new-build-system/user-guide/manifest-merger#TOC-Placeholder-support)（*manifest-merger也是Android Plugin for Gradle的一项功能*）功能一起，实现对manifest里面的渠道号的控制，从而实现正对多种渠道的打包，在持续集成应用上，build types也能用来同时生成debug和release版本。

多渠道打包build.gradle：
```groovy
android {
    defaultConfig {
        minSdkVersion 8
        versionCode 10
    }
    productFlavors {
        YingYongBao {
			manifestPlaceholders = [CHANNEL_ID = "111"]
        }
        XiaoMi {
	        manifestPlaceholders = [CHANNEL_ID = "222"]
        }
    }
    buildTypes {
	    debug {
		    applicationId = "com.xx.xx.debug"
	    }
	    release {
		    applicationId = "com.xx.xx"
	    }
    }
}
```

Placeholders技术，在manifest文件中默认有一个“applicationId”的place holder，对应在manifest文件中的
```
<activity  android:name=".Main">
	<meta-data android:name="channel_id" android:value="${CHANNEL_ID}"/>
	<intent-filter>
       <action android:name="${applicationId}.foo"></action>
    </intent-filter>
</activity>
```
按照上面的build.gradle debug版本的配置，将会被改成
```
    ...
	<meta-data android:name="channel_id" android:value="111"/>
	...
		<action android:name="com.xx.xx.debug.foo"></action>
```
其他是自定义格式的place holder，例如上面的"CHANNEL_ID"，基本上就是一个值替换的过程。只不过写法上没有默认的place holder这么直观。


#### Dependencies依赖关系
Android plugin for Gradle支持以下几种方式的依赖处理
* 模块依赖(module dependencies)
* 本地依赖(local dependencies)
* 远程依赖(remote dependencies)
```
dependencies {
	compile fileTree(dir: 'libs', include: '*.jar')
	compile files('libs/foo.jar')  //本地依赖
    compile 'com.google.guava:guava:11.0.2' //远程依赖
}
```
#### Build tasks构建任务
默认有几个task构建任务
* assemble - Builds the project output.
* check - Runs checks and tests.
* build - Runs both assemble and check.
* clean - Performs the clean.

#### 源代码文件目录
To build each version of your app, the build system combines source code and resources from:

* src/main/ - the main source directory (the default configuration common to all variants)
* src/buildType/ - the source directory
* src/productFlavor/ - the source directory

For example, to generate the default debug and release build variants in projects with no product flavors, the build system uses:

* src/main/ (default configuration)
* src/release/ (build type)
* src/debug/ (build type)


## Sample
```
apply plugin: 'com.android.application'

def releaseTime() {
    return new Date().format("yyyy-MM-dd", TimeZone.getTimeZone("UTC"))
}

android {
    compileSdkVersion 21
    buildToolsVersion '21.1.2'

    defaultConfig {
        applicationId "com.boohee.*"
        minSdkVersion 14
        targetSdkVersion 21
        versionCode 1
        versionName "1.0"
        
        // dex突破65535的限制
        multiDexEnabled true
        // 默认是umeng的渠道
        manifestPlaceholders = [UMENG_CHANNEL_VALUE: "umeng"]
    }

    lintOptions {
        abortOnError false
    }

    signingConfigs {
        debug {
            // No debug config
        }

        release {
            storeFile file("../yourapp.keystore")
            storePassword "your password"
            keyAlias "your alias"
            keyPassword "your password"
        }
    }

    buildTypes {
        debug {
            // 显示Log
            buildConfigField "boolean", "LOG_DEBUG", "true"

            versionNameSuffix "-debug"
            minifyEnabled false
            zipAlignEnabled false
            shrinkResources false
            signingConfig signingConfigs.debug
        }

        release {
            // 不显示Log
            buildConfigField "boolean", "LOG_DEBUG", "false"

            minifyEnabled true
            zipAlignEnabled true
            // 移除无用的resource文件
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            signingConfig signingConfigs.release

            applicationVariants.all { variant ->
                variant.outputs.each { output ->
                    def outputFile = output.outputFile
                    if (outputFile != null && outputFile.name.endsWith('.apk')) {
                    	// 输出apk名称为boohee_v1.0_2015-01-15_wandoujia.apk
                        def fileName = "boohee_v${defaultConfig.versionName}_${releaseTime()}_${variant.productFlavors[0].name}.apk"
                        output.outputFile = new File(outputFile.parent, fileName)
                    }
                }
            }
        }
    }

    // 友盟多渠道打包
    productFlavors {
        wandoujia {}
        _360 {}
        baidu {}
        xiaomi {}
        tencent {}
        taobao {}
        ...
    }

    productFlavors.all { flavor ->
        flavor.manifestPlaceholders = [UMENG_CHANNEL_VALUE: name]
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.android.support:support-v4:21.0.3'
    compile 'com.jakewharton:butterknife:6.0.0'
    ...
}
```
