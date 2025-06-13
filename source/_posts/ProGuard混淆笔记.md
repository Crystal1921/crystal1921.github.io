---
title: ProGuard混淆笔记
date: 2025-6-13 22:28:00
tags: 
  - java
  - mod
  - mc
---

使用ProGuard对mc模组进行混淆时，有许多需要注意的问题。

首先需要准备两个文件，`dic.txt`用来储存混淆的字典表，用来做为混淆后的像这样的字符串`oo00O0O0`，`proguard.pro`用来储存ProGuard的规则来。

为了混淆jar文件，需要新建一个`proguard`任务

``` groovy
tasks.register('proguard', ProGuardTask) {
    dependsOn tasks.jar

    // 指定配置文件
    configuration files('proguard.pro')

    // 运行时类路径
    configurations.compileClasspath.getResolvedConfiguration().getFiles().each { file ->
        if (file.isFile() && file.name.endsWith('.jar')) {
            libraryjars file.path
        }
    }

    // 输入输出
    injars tasks.jar.outputs.files.singleFile
    outjars layout.buildDirectory.file("libs/${archivesBaseName}-minified.jar")
}
```

这里我们使用原来nf的jar任务构建出一个新的模组文件作为输入，经过ProGuard混淆产生新的被混淆的jar文件

注意这里需要使用compileClasspath编译时环境作为jar库，不能使用runtimeClasspath，因为像Lombok，IDEA注解等只有在编译环境存在。

下面来到了最核心的部分，ProGuard规则文件`proguard.pro`

``` ProGuard
# 保留规则
# 保留所有 Lombok 注解和生成的代码
-keep class lombok.* { *; }
-keep @interface lombok.*
-keepclassmembers class * {
    @lombok.* *;
}

# 保持类不被混淆
-keep class com.example.examplemod.MODID { *; }
-keep,allowobfuscation class com.example.examplemod.** { *; }
-keep class com.example.examplemod.mixin.** { *; }

-keepclasseswithmembers,allowobfuscation,includedescriptorclasses class * {
    @com.google.gson.annotations.SerializedName <fields>;
}

# 属性保留
-keepattributes Exceptions, InnerClasses, Signature, Deprecated, LineNumberTable, *Annotation*, EnclosingMethod, EventHandler, Override, Synthetic

# 优化配置
-dontoptimize  # 禁用所有优化
-ignorewarnings
-verbose

# 混淆字典
-obfuscationdictionary dic.txt
-classobfuscationdictionary dic.txt
-packageobfuscationdictionary dic.txt

# 重打包
-repackageclasses 'com.example.examplemod'

# 映射文件
-printmapping build/proguard/mappings.map

# 库文件
-libraryjars <java.home>/jmods/java.base.jmod(!**.jar,!module-info.class)
-libraryjars <java.home>/jmods/java.net.http.jmod(!**.jar,!module-info.class)
-libraryjars <java.home>/jmods/java.desktop.jmod(!**.jar,!module-info.class)
```

这里需要注意的是由于Java9以后的模块化，需要按模块导入Java模块。

这里针对使用Lombok生成的代码进行特判，并且保持了用于GSON序列化字段的`SerializedName`注解。

这里需要注意的是，由于ProGuard会对字段进行混淆，混淆后与原来字段不符合，会导致gson类初始化失败，这时候必须显示使用`@SerializedName`注解标注每个字段，用来确保混淆后仍然可以正确序列化json文件。

在`keepattributes`中，保留了注解，内部类等信息，因为mc的主类，事件监听器都是通过注解处理器实现的，所以必须保留这些内容。