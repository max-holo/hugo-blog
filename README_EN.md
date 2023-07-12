Language: English | [Chinese](README.md)



> Hugo version 0115.2, this topic according to Hugo PaperMod theme change to: https://github.com/adityatelange/hugo-PaperMod



### [preview](https://www.loadingspace.cn/)



## 1. git clone pulls code



① Use 'git clone' to pull the code



② Go to the hugo-blog directory and enter 'git submodule update --init', which means pull the submodule under themes/hugo-PaperMod/ and put the official themes init



## 2. Start the screen



③ Locate the directory under Hugo-blog, enter 'hugo server -D' in the terminal, and enter: localhost:1313 in the browser to see the ready-made blog template.



## 3. Modify the information



The template contains a lot of personal information that needs to be configured by yourself. Please modify it patiently



## 4. Usage of shortcodes

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


## 5. Possible problems



1. Some users will be deployed to github, may encounter cross-system problems, such as "LF will be replaced by CRLF in ******", then enter the command: 'git config core. lf false' to solve the problem of automatic conversion of newlines.

2.0111.3 in hugo version config.yaml
params:
profileMode:
Need to remove params instead:
