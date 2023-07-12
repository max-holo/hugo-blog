语言：中文 | [English](README_EN.md)

> hugo版本0115.2,该主题根据Hugo PaperMod主题修改而来: https://github.com/adityatelange/hugo-PaperMod

### [预览](https://www.loadingspace.cn/)

## 1. git clone 拉取代码

① 用`git clone`的方式拉取代码

② 进入到hugo-blog目录，输入`git submodule update --init`，表示拉取themes/hugo-PaperMod/下的子模块，里面放的是官方主题

## 2. 启动界面

③ 把目录定位到hugo-blog下，在终端输入`hugo server -D`，在浏览器输入：localhost:1313 即可看到现成的博客模板。

## 3. 修改信息

模板内部有许多个人信息需要自己配置，请耐心修改完

## 4. shortcodes使用方法

`bilibili: {{< bilibili BV1Fh411e7ZH(填 bvid) >}}`

`youtube: {{< youtube w7Ft2ymGmfc >}}`

`ppt: {{< ppt src="网址" >}}`

`douban: {{< douban src="网址" >}}`

```
# 文章内链卡片
# 末尾要加 md，只能填写相对路径，如下
{{< innerlink src="posts/tech/mysql_1.md" >}}
```

```
gallery:

{{< gallery >}}
{{< figure src="https://www.sulvblog.cn/posts/read/structural_thinking/0.png" >}}
{{< figure src="https://www.sulvblog.cn/posts/read/structural_thinking/0.png" >}}
{{< /gallery >}}
```

## 5. 可能遇到的问题

1. 有些使用者会部署到github，可能遇到跨系统的问题，如提示`LF will be replaced by CRLF in ******`，这时输入命令：`git config core.autocrlf false`，解决换行符自动转换的问题。
2. 0111.3以下的hugo版本 config.yaml中
   params:
   profileMode:
   需要改为去掉params:
    