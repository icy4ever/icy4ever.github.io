## Git命令相关

安装

```bash
apt-get install git
```

#### Clone

克隆最新版本到libgit2目录下

```bash
$ git clone https://github.com/libgit2/libgit2
```

也可以指定目录

```bash
$ git clone https://github.com/libgit2/libgit2 /usr/local
```

#### Status

查看当前分支状态

```bash
$ git status
On branch user_tags_2
Your branch is up to date with 'origin/user_tags_2'.

nothing to commit, working tree clean
```

显示简介

```bash
$ git status -s 
 M .idea/workspace.xml
 M "md/Git\345\221\275\344\273\244\347\233\270\345\205\263.md"
A  test.text
?? down.txt
```

M-修改

A-新增文件（已暂存） 

??-代表未追踪

可以多个组合，譬如新增一个文件并修改了，那左边是 AM。

#### Add

追踪新文件

如果是文件夹 那么使用递归的方式去遍历文件夹下所有文件

```bash
$ git add test.txt
```

#### Ignore

如果期望忽略文件，请使用以下方法

在跟目录新建.gitignore

```
1. 空行和#开头的行将被忽略
2. 这个文件将在整个文件树中递归的遍历文件并忽略那些与之匹配的
3. 你可以指定目录或整个目录下的文件，使用（ “/” ）绝对路径
4. 使用！来做否定语法
```

#### Diff

以下命令会展示简略的变化

```bash
$ git diff --stat
```

对比两个分支上的最新提交

```bash
$ git diff submit1 submit2
```

单对比当前分支和另外一个指定分支的差别

```bash
$ git diff master
```

对比当前目录的lib文件夹和当前分支的上次提交之间的差别

```bash
$ git diff HEAD -- ./lib
```

对比上次提交和上上次提交之间的差别（^可以多个）

```bash
$ git diff HEAD^ HEAD
```

根据版本SHA1值做对比

```bash
$ git diff SHA1 SHA2
```

对比当前分支的上次提交和上上次提交的某个目录的差别

```bash
$ git diff head^ head ./articleHtml
```

#### Commit

提交，这很简单

```bash
$ git commit
```

直接提交所有更改，不需要事先git add

```bash
$ git commit -a -m 'change sth'
```

#### RM

基本删除（停止追踪并且在磁盘中删除此文件）

```bash
$ git rm test.txt
```

保留文件在磁盘上但是停止追踪此文件

```bash
$ git rm --cached test.text
```

删除远端分支

```bash
$ git push origin --delete branch_name 
```

#### MV

这相当于一个重命名操作

```bash
$ git mv read.me read
```

等同于以下操作

```bash
$ mv read.me read
$ git rm read.me
$ git add read
```



#### 记住密码

因为在远端linux总是需要拉最新代码，所以创建自动化部署脚本之后总是要输入密码，

这里提供解决方法之一：

```bash
sudo vim  项目根目录/.git/config
```

在文件底部增加

```bash
[credential]
        helper = store
```

以此在输入过密码后可以记住，无需再次输入。