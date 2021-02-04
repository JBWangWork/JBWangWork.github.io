---
title: Mac电脑配置福利Alfred、Go2shell、iTerm2+Oh My Zsh
date: 2019-03-05 12:44:39
author: Vincent
categories: 
- 软件
tags: 
- Mac配置
---

![电脑配置](https://upload-images.jianshu.io/upload_images/5741330-d655dfa497bbcfd7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


哎，一年换了个21-inch iMac，两个27-inch iMac，加上重做系统就更不说了，每次都要下载各种软件，各种配置。。。故记录这篇文章以免自己以后老了记不住，希望可以帮到更多人吧！

####  效率神器Alfred、Go2shell
首先，拿到新电脑，需要下载很多软件，第一时间把`Alfred`和`Go2shell`安装好，这里有[各种破解软件免费下载平台](xclient.info)，里面安装教程很详细，默认`alt + 空格`打开Alfred，Go2shell就是可以快速打开当前路径的终端，用起来还是很方便的。
![Go2shell](https://upload-images.jianshu.io/upload_images/5741330-8400d17b9f794546.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 终端配置
接下来根据个人喜好配置终端，之前也一直在用苹果自带的终端，但是自从接触了iTerm2后就无法自拔了。
- 安装iTerm2
 先去官网下载iTerm2，[iTerm2下载地址：](http s://www.iterm2.com/downloads.html)[https://www.iterm2.com/downloads.html](https://www.iterm2.com/downloads.html)

- 配置iTerm2
目前iTerm2 最常用的主题是` Solarized Dark theme`，下载地址：[http://ethanschoonover.com/solarized](http://ethanschoonover.com/solarized)，下载好的压缩文件解压后，打开 iTerm2的`Preferences `配置界面，可以按`Command + ,`键打开 ，然后`Profiles -> Colors -> Color Presets -> Import`

![添加主题文件](https://upload-images.jianshu.io/upload_images/5741330-889dc79aadf6e185.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

选择刚才解压的solarized->iterm2-colors-solarized->Solarized Dark.itermcolors文件，导入成功，最后选择 Solarized Dark 主题。

#### 配置Oh My Zsh
一款用于管理zsh配置，可以提供超级多而强大的插件和漂亮的主题。可以去GitHub下载[Oh My Zsh](https://github.com/robbyrussell/oh-my-zsh)
- 安装Oh My Zsh
```
// 使用 crul 安装：
$ sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```
或者
```
// 使用wget：
$ sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
```
安装成功后

![配置Oh My Zsh](https://upload-images.jianshu.io/upload_images/5741330-3fe231151c615c15.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 卸载Oh My Zsh
```
// 卸载oh-my-zsh命令：
$ uninstall_oh_my_zsh
```
- 设置当前用户的默认Shell为Zsh
```
$ chsh -s /bin/zsh
```
接下来，将主题配置修改为ZSH_THEME="agnoster"
```
$ vim ~/.zshrc
```
到目前为止，不出意外的话，iTerm2外观应该是这样的

![iTerm2外观](https://upload-images.jianshu.io/upload_images/5741330-92007dc6e28a9a86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

界面显示乱码原因是没有安装`Meslo`字体，字体下载地址：[Meslo LG M Regular for Powerline.ttf](https://github.com/powerline/fonts/blob/master/Meslo%20Slashed/Meslo%20LG%20M%20Regular%20for%20Powerline.ttf)，下载后安装接下来还是打开 iTerm2的Preferences 配置界面，可以按`Command + ,`键打开 ，然后`Profiles -> Text -> Font -> Chanage Font`

![配置Meslo字体](https://upload-images.jianshu.io/upload_images/5741330-ffed44579f071798.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
成功后的截图
![iTerm2+Oh My Zsh](https://upload-images.jianshu.io/upload_images/5741330-e95f6971f45f9469.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

也可以修改合适的字体大小。
#### 自动填充
这个功能是非常实用的，可以方便我们快速的敲命令。

配置步骤，先克隆`zsh-autosuggestions`项目，到指定目录：
```
$ git clone https://github.com/zsh-users/zsh-autosuggestions ~/.oh-my-zsh/custom/plugins/zsh-autosuggestions
```
然后编辑`vim ~/.zshrc`文件，找到`plugins`配置，增加`zsh-autosuggestions`插件。如果配置不生效增加zsh-syntax-highlighting插件试试。

![自动填充](https://upload-images.jianshu.io/upload_images/5741330-275b4a55100f9dac.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 隐藏用户名和主机名
进入Oh My Zsh主题文件列表
```
$ cd ~/.oh-my-zsh/themes
```
进入已选的主题，并找到`prompt_context`，然后进行修改
```
### Prompt components
# Each component will draw itself, and hide itself if no information needs to be shown

# Context: user@hostname (who am I and where am I)
prompt_context() {
  if [[ "$USER" != "$DEFAULT_USER" || -n "$SSH_CLIENT" ]]; then
    # prompt_segment black default "%(!.%{%F{yellow}%}.)%n@%m"
    prompt_segment black default "Vincent" // Vincent是写死的名字 可以根据个人爱好随意设置
  fi
}
```
![隐藏用户名和主机名](https://upload-images.jianshu.io/upload_images/5741330-3c6300f5a99d9272.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

还有一些插件和功能网上很多，暂不做更多介绍。

该文章为记录本人的电脑配置，希望能够帮助大家！！！

