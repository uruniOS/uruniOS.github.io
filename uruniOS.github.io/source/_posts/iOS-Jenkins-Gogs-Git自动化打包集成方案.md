---
title: iOS-Jenkins-Gogs-Git自动化打包集成方案
date: 2017-01-23 15:30:52
tags: jenkins、自动化、git
categories: 技术
author: yfm
---

## 前言
放弃手动打包，拥抱自动化

## 搭建Jenkins环境

[下载Jenkins.war安装包](https://jenkins.io/index.html)
[下载JDK](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)

有两个版本：LTS Release、Weekly Release
我选择的是LTS Release

下载完成之后，放到合适的目录，进入目录，执行

<!--more-->


```
java -jar jenkins.war
```
启动Jenkins，jenkins默认使用8080端口启动，如果启动失败，可使用自定义端口号启动，命令如下

```
java -jar jenkins.war --httpPort=7080
```

## 安装插件
**SCM API Plugin** 提供API上传插件的工具
**Keychains and Provisioning Profiles Management** 用于管理钥匙串以及PP文件的插件
**Git plugin** Git插件
**Gogs plugin** Gogs插件，用来触发webhooks
**SSH plugin**  通过ssh协议远程执行shell的插件
**Email Extension Plugin** 自定义邮件格式插件

## 配置Jenkins

**配置证书文件**

Jenkins主界面->系统管理->Keychains and Provisioning Profiles Management

* 添加钥匙串

点击选择文件，然后选择自己存放分发证书的钥匙串，一般来说是钥匙串文件保存在

```
/Users/developer/Library/Keychains/login.keychain
```

上传后在Password处输入解锁钥匙串的密码，这个密码是账户的管理员密码

然后添加Code Signing Identify。这个值一般来说进入自己的钥匙串->右键需要添加的证书->显示介绍

然后把对于的Code Signing Identify的名字复制粘贴出来就可以了

* 添加PP证书

PP文件的地址

```
~/Library/MobileDevice/Provisioning Profiles/
```
 
选择对应的PP文件，一般选择团队PP证书（Team_Provisioning_profiles.mobileprovision），然后Upload即可。这里有一点注意的是，最好*.mobileprovision的文件名最好与UUID相同 (不相同会有问题)

设置完成之后如下图：
![D4CB61D8-C4B6-43A6-A68E-3AE73D3BCEB7](/media/D4CB61D8-C4B6-43A6-A68E-3AE73D3BCEB7.png)￼



**配置邮箱**
**配置邮件提醒**
这里以QQ邮箱为例

登录管理员邮箱，打开首页->设置->账户->开启POP3/SMTP服务->发送短信

![](/media/14818558798311.jpg)￼


发送短信完成之后点击我已发送，会收到QQ回执的密码，这个密码下面会用到

打开Jenkins->系统设置-> Extended E-mail Notification->打开高级

设置
SMTP server: smtp.qq.com
Default user E-mail suffix: @qq.com

勾选Use SMTP Authentication
设置
用户名：管理员邮箱名
密码：上面QQ回执的密码
SMTP端口:465
字符集：UTF-8

设置完成，测试邮件配置
勾选通过发送测试邮件测试配置，填写邮箱地址，点击Test configuration
发送成功会显示：	
*Email was successfully sent*

![15D74041-0FD2-474D-A3AC-84327B421293](/media/15D74041-0FD2-474D-A3AC-84327B421293.png)￼



## 创建项目
打开jenkins首页，点击创建->jenkinsTest->构建一个自由风格的软件项目->OK,创建成功

## 设置webhooks

以Gogs为例，webhooks格式如下

```
http(s)://<< jenkins-server >>/gogs-webhook/?job=<< jobname >>
```

打开项目Gogs地址->仓库设置->管理web钩子
设置推送地址：

```
http://{你的jenkins服务器地址}/gogs-webhook/?job=JenkinsTest
```

数据格式

```
application/x-www-form-urlencoded
```

![7F3DC869-201F-4778-9666-99A1E2877A09](/media/7F3DC869-201F-4778-9666-99A1E2877A09.png)￼



## 配置项目

### General

勾选**丢弃旧的构建**
设置：
最大保留天数2
最大构建数5

### 源码管理
设置：
Repository URL: http://{Gogs地址}/{项目名}/jenkinsTest.git
Credentials: gogs账号用户名和密码
Branch: */master

### 构建环境

选择Keychains and Code Signing Identities，Jenkins会自动选择自之前配置的证书

![5A87EA35-C429-4890-A1D0-276E4876363E](/media/5A87EA35-C429-4890-A1D0-276E4876363E.png)￼



### 构建

选择Execute shell构建，构建脚本如下:

```
# 注意：shell变量和等号之间不能有空格
# exit 0 成功
# exit 1（非0的返回状态码） 失败

# 获取Info.plist文件路径
Info_plist_dir="${WORKSPACE}/${JOB_NAME}/Info.plist"

# 获取CI.plist文件路径
CI_plist_dir="${WORKSPACE}/${JOB_NAME}/CI.plist"

# 获取CI.plist文件打包环境
IS_Debug=$(/usr/libexec/PlistBuddy -c "Print :IS_DEBUG_ARCHIVE" ${CI_plist_dir})

# 获取CI.plist文件build和release版本号
IS_Need_archive=$(/usr/libexec/PlistBuddy -c "Print :IS_NEED_ARCHIVE" $CI_plist_dir)

if [[ ! -n "$IS_Debug" || ! -n "$IS_Need_archive" ]]; then

    echo "打包环境设置不完整"
    exit 1
    
fi

if [[ "$IS_Need_archive" = false ]]; then

    echo "不需要打包"
    exit 1
    
fi

# 获取Info.plist文件version和build版本号 CFBundleShortVersionString CFBundleVersion
bundle_version=$(/usr/libexec/PlistBuddy -c "Print :CFBundleShortVersionString" ${Info_plist_dir})
bundle_build_version=$(/usr/libexec/PlistBuddy -c "Print :CFBundleVersion" ${Info_plist_dir})

echo "xcode文件version版本号: $bundle_version"
echo "xcode文件build版本号: $buidle_build_version"

# 设置xcarchive导出路径
debug_xcarchive_out_path="/Users/rurn/DailyBuild/测试工程/测试包/xcarchive/${JOB_NAME}_v${bundle_build_version}_$(date +%F)_alpha.xcarchive"
release_xcarchive_out_path="/Users/rurn/DailyBuild/测试工程/正式包/xcarchive/${JOB_NAME}_v${bundle_version}_$(date +%F)_release.xcarchive"


# 设置ipa导出路径
debug_ipa_out_path="/Users/rurn/DailyBuild/测试工程/测试包/ipa/${JOB_NAME}_v${bundle_build_version}_$(date +%F)_alpha.ipa"
release_ipa_out_path="/Users/rurn/DailyBuild/测试工程/正式包/ipa/${JOB_NAME}_v${bundle_version}_$(date +%F)_release.ipa"

# 根据CI.plist文件设置的环境进行打包

if [[ "$IS_Debug" = true ]]; then
#Debug环境打包
    echo "打包环境:Debug"
    if [[ "$IS_Need_archive" = false ]]; then
    
        echo "不需要打包"
        exit 1
        
    else
    
        echo "打debug包"
        
        if [[ -e $debug_xcarchive_out_path || -e $debug_ipa_out_path ]]; then
            echo "同名文件以及存在，请先修改xcode文件build版本号后再次尝试！"
            exit 1
        fi
        
        # 清空上次构建
        xcodebuild clean
        
        # 打包
        xcodebuild archive -workspace ./JenkinsTest.xcworkspace/ -scheme JenkinsTest -archivePath ${debug_xcarchive_out_path} -configuration Debug -sdk iphoneos

        # 导出安装包
        xcodebuild -exportArchive -archivePath ${debug_xcarchive_out_path} -exportPath ${debug_ipa_out_path} -exportFormat ipa -exportProvisioningProfile 'JenkinsTestAdHoc'

    fi

else
#Release环境打包
    echo "打包环境:Release"
    
    if [[ "$IS_Need_archive" = false ]]; then
    
        echo "不需要打包"
        exit 1
        
    else
    
        echo "Release包"
        
        if [[ -e $release_xcarchive_out_path || -e $release_ipa_out_path ]]; then
            echo "同名文件以及存在，请先修改xcode文件version版本号后再次尝试！"
            exit 1
        fi

        # 清空上次构建
        xcodebuild clean

        # 打包
        xcodebuild archive -workspace ./JenkinsTest.xcworkspace/ -scheme JenkinsTest -archivePath ${release_xcarchive_out_path} -configuration Release -sdk iphoneos

        # 导出安装包
        xcodebuild -exportArchive -archivePath ${release_xcarchive_out_path} -exportPath ${release_ipa_out_path} -exportFormat ipa -exportProvisioningProfile 'JenkinsTestAdHoc'

    fi
fi
```

**设置触发条件：**
Triggers：
Success（脚本构建成功，正常退出(exit 0)）
Send To: Developers (管理员)、Recipient List （开发者列表）

Fail（脚本构建失败，异常退出（exit 非0，如：exit 1））

Always (总是发送：)
Developers（管理员）

![](/media/14818709984719.jpg)￼


### 构建后操作

设置邮件提醒

选择Editable Email Notification

Project Recipient List： 接收人邮件，多个用逗号分隔
Project Reply-To List：回复邮件，设置默认地址$DEFAULT_REPLYTO
Content Type: HTML
Default Content

```
(本邮件是程序自动下发的，请勿回复！)<br/><hr/>

项目名称：$PROJECT_NAME<br/><hr/>

构建编号：$BUILD_NUMBER<br/><hr/>

构建状态：$BUILD_STATUS<br/><hr/>

触发原因：${CAUSE}<br/><hr/>

构建日志地址：<a href="${BUILD_URL}console">${BUILD_URL}console</a><br/><hr/>

构建地址：<a href="$BUILD_URL">$BUILD_URL</a><br/><hr/>

安装包地址：<a> http://{你的jenkins服务器地址}/DailyBuild/测试工程</a><br/><hr/>

变更集:${JELLY_SCRIPT,template="html"}<br/><hr/>

```


## 构建测试

1、设置打包环境为Debug
打开CI.plist文件设置`IS_DEBUG_ARCHIVE`为true

2、设置`IS_NEED_ARCHIVE`为true

3、设置Info.plist文件Build版本号，**不能与上次构建的相同，否则会打包失败，因为导出的目录已经存在一个同名的文件**

4、提交代码到仓库，完成自动化打包

5、查看邮件

