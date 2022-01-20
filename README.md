# Gradle 模块化、工程化管理

> Gradle在android日常开发工作中无处不在,可能你对Gradle原理不太清楚，但是快速掌握它的使用技巧，将为你带来事倍功半的效果。



## Gradle

- 通俗的讲它是基于JVM的构建工具，通过Groovy的脚本构建任务，并管理任务与任务之前的依赖

- 构建生命周期三阶段，当然可以做监听这里我们不展开

  - 初始化阶段：解析`setting.gradle`文件，对参与构建的项目创建project对象

  - 配置阶段：解析每个project的`build.gradle`,完成以下任务

    - 加载插件 

      ```
      //例如
      plugins {
          id 'com.android.application'
      }
      ```

    - 加载依赖

      ```
      //例如
      dependencies {
      	...
      }
      ```

    - 加载Task

    - 执行脚本，自定义插件、Gradle支持的API等

  - 执行阶段：执行加载的Task以及依赖的Task


## 循环优化`Gradle`依赖管理

一般情况下我们都是直接在build.gradle文件中定义依赖

```
dependencies {
    implementation 'androidx.appcompat:appcompat:1.2.0'
    implementation 'com.google.android.material:material:1.3.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.0.4'
    implementation 'androidx.navigation:navigation-fragment:2.3.5'
    implementation 'androidx.navigation:navigation-ui:2.3.5'
    testImplementation 'junit:junit:4.+'
    androidTestImplementation 'androidx.test.ext:junit:1.1.2'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.3.0'
}
```

但是随着项目中越来越大，如何管理上百上千个依赖？这里我们就要借助到Groovy的`List`

root目录新建config文件夹，在config文件夹中新建`version.gradle，将配置相关的文件都放到项目的config文件夹`统一管理，这里在ext中添加了函数`ext.depen`

```
ext {
    appcompat="1.2.0"
    material="1.2.0"
    constraintlayout="2.0.4"
    navigation_fragment="2.3.5"
    navigation_ui="2.3.5"
    junit="4.+"
    ext_junit="1.1.2"
    espresso_core="3.3.0"
    glide="4.8.0"
    androidx_core="1.3.2"
}
ext.depen = { android,obj ->
    android.each { entry ->
        if(entry.value instanceof Map){
            obj.api (entry.key){
                entry.value.each { childEntry ->
                    exclude(group:childEntry.key)
                    exclude(module:childEntry.value)
                }
            }
        }else {
            obj.api entry.value
        }

    }
}
```

新建`dependencies.gradle`

```
apply from: "../config/version.gradle"

ext {
    android = [
        "appcompat":"androidx.appcompat:appcompat:${appcompat}",
        "constraintlayout":"androidx.constraintlayout:constraintlayout:${constraintlayout}",
        "androidx.navigation:navigation-ui:${navigation_ui}" : [
                'androidx.navigation' : 'navigation-runtime',
                'androidx.customview' : 'customview',
                'androidx.drawerlayout' : 'drawerlayout',
                'com.google.android.material' : 'material'
        ],
        "navigation_fragment":"androidx.navigation:navigation-fragment:${navigation_fragment}",
    ]

    test = [
            "junit":"junit:junit:${junit}"
    ]

    androidTest = [
            "ext_junit":"androidx.test.ext:junit:${ext_junit}",
            "espresso_core":"androidx.test.espresso:espresso-core:${espresso_core}"
    ]
}
```

新建config.gradle,**Gradle模块化**，开发过程可以针对一些单独或则通用的配置单独的抽取，通过**apply from .....**引入

```
apply from: "../config/dependencies.gradle"

android {
    compileSdk 31

    defaultConfig {
        applicationId "com.tip.mode1"
        minSdk 21
        targetSdk 31
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}

dependencies {
    def android = project.ext.android
    def test = project.ext.test
    def androidTest = project.ext.androidTest

    depen(android,getDelegate())

    test.each {k,v -> testImplementation v}
    androidTest.each {k,v -> androidTestImplementation v}
}

```

然后在每个project的build.gradle中添加即可，这样做的优点是能保持每个project简洁切修改方便

```
......
apply from: "../config/config.gradle"
```

## 代码提示的Gralde

这里我们采用```自定义 kotlin+plugin + includeBuild```管理，构建速度更快一些,支持自动补全和单机跳转，方便你更好的管理Gradle依赖

新建 module ：version

修改 build.gradle

```
apply plugin: 'kotlin'
apply plugin: 'java-gradle-plugin'

buildscript {
    repositories {
        jcenter()
        google()
    }
    dependencies {
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:1.6.10"
    }
}

repositories {
    jcenter()
    google()
}

dependencies {
    implementation gradleApi()
    implementation "org.jetbrains.kotlin:kotlin-gradle-plugin:1.6.10"
}

compileKotlin {
    kotlinOptions {
        jvmTarget = "1.8"
    }
}
compileTestKotlin {
    kotlinOptions {
        jvmTarget = "1.8"
    }
}

gradlePlugin {
    plugins {
        version {
            id = 'com.tip.version'
            implementationClass = 'com.tip.version.VersionPlugin'
        }
    }
}

```

增加kt

```
import com.tip.Compileimport com.tip.Googleplugins {    id 'com.android.application'    id "com.tip.version"}apply from: "../config/config.gradle"android {    compileSdk Compile.minSdkVersion		......}dependencies {    implementation Google.material    ......}    
```



##  资源分包

通过gradle的**sourceSets**属性

```
android {    //...    sourceSets {        main {            res.srcDirs(                    'src/main/res',                    'src/main/res_core',                    'src/main/res_user',            )        }    }}
```



## 源码依赖与aar远程快速切换

我们在模块化过程中需要发布aar集成到主app，但是我们常常需要修改代码进行源码依赖，我们需要快速实现切换

```
include ':glide-source'project(':glide-source').projectDir = new File("../../lib/retrofit-source")
```



```
configurations.all {    resolutionStrategy {        dependencySubstitution {            substitute module( "com.github.bumptech:glide") with project(':glide-source')        }    }}
```

