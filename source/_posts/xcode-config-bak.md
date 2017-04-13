title: Xcode一些配置记录
date: 2015-10-26 22:09:59
tags: [iOS, Xcode]
categories: Xcode
---
记录一下Xcode经常会用到的一些配置信息
## 使用svn号作为build号

在Build Phase---Run Srcipt添加以下脚本
``` bash
REV=`svnversion -nc | /usr/bin/sed -e 's/^[^:]*://;s/[A-Za-z]//'`
BASEVERNUM=`/usr/libexec/PlistBuddy -c "Print :CFBundleShortVersionString" "${INFOPLIST_FILE}"`
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion $REV" "${INFOPLIST_FILE}"
```
## Xcode插件失效解决方案
<!-- more -->
### Xcode版本更新后插件失效
每次Xcode版本更新后已安装插件都会失效，是由于Xcode UUID进行了修改，所以要更新所有插件支持的UUID

- 首先使用以下命令查看当前版本UUID
``` bash
defaults read /Applications/Xcode.app/Contents/Info DVTPlugInCompatibilityUUID
```
- 使用获取的UUID更新插件配置文件
``` bash
find ~/Library/Application\ Support/Developer/Shared/Xcode/Plug-ins -name Info.plist -maxdepth 3 | xargs -I{} defaults write {} DVTPlugInCompatibilityUUIDs -array-add 获取的UUID
```

### Xcode提示load boundle时误点skip boundle后插件被禁用
- 可以使用以下命令查看插件是否被禁用(7.1为例)
``` bash
defaults read com.apple.dt.Xcode DVTPlugInManagerNonApplePlugIns-Xcode-7.1
```

- 使用以下命令删除这个配置，下次启动时就会重新询问了
``` bash
defaults delete com.apple.dt.Xcode DVTPlugInManagerNonApplePlugIns-Xcode-7.1
```

