---
title: 如何整理和分析commit
date: 2019-07-16 23:16:21
tags: commit analysis
---



在同步代码或者升级社区开源组件的版本时候经常需要分析不同分支之间的差异，或者分析某一个开源组件中的自验代码的情况，这时候可以通过git工具来批量导出所有commit。



<!-- more -->

## 1 一条命令导出当前分支所有commit



在当前代码库目录的当前分支执行以下命令即可将所有commit导出到`/home/commits.csv`文件中：

```shell
git log --no-merges 721bfa7..HEAD --date=iso --pretty=format:'%h|%s|%an|%ad|' --shortstat | sed '/^$/d' | sed -n '1h;1!H;${g;s/|\n/,/g;p;}' > /home/commits.csv
```



其中：

- `--no-merges`意味着去掉`Merge`的commit
- `721bfa7..HEAD`是commit的范围，其中`21bfa7`是起始commit之前的commit id，`HEAD`是最新这个commit，`721bfa7..HEAD`的意思是这个区间内不包含`721bfa7`的所有commit的总和，所以在执行这个命令之前一定要先查清楚起始的commit是哪个，然后使用这个commit之前的那个commit id（我查到一些资料说`721bfa7 HEAD`就能取到包含`721bfa7`的列表，但是试了试没有成功）
- `--date=iso`意味着使用iso标准来输出时间
- `--pretty=format:`用于指定要输出哪些内容，其中之所以使用`|`来作为分隔符主要是因为很多标题中包含`,`，后期不好处理，`|`存在于标题中的概率较小，在使用时也可以改成其他分隔符
  - `%h`：提交对象的简短哈希字串
  - `%s`：提交说明（commit的标题）
  - `%an`：作者（author）的名字
  - `%ad`：作者修订日期（可以用 -date= 选项定制格式）
- `--shortstat`用于输出每个commit中的修改文件个数、新增代码量和减少代码量
- `sed '/^$/d'`用于删除空行
- `sed -n '1h;1!H;${g;s/|\n/,/g;p;}'`用于合并commit信息（`'%h|%s|%an|%ad|'`）和`--shortstat`到同一行



有个`commits.csv`就可以接下来继续整理做表格了

- commit id列

```shell
cat  /home/commits.csv | cut -d '|' -f 1
```

- commit名称列

```shell
cat  /home/commits.csv | cut -d '|' -f 2
```

- commiter列

```shell
cat  /home/commits.csv | cut -d '|' -f 3
```

- commit时间列

```shell
cat /home/commits.csv | cut -d '|' -f 4 | cut -d ',' -f 1 
```

- 修改文件数量列

```shell
cat /home/commits.csv | cut -d '|' -f 4 | cut -d ',' -f 2 | cut -d ' ' -f 2 
```

- 新增代码数量列

```shell
cat /home/commits.csv | cut -d '|' -f 4 | cut -d ',' -f 3 | cut -d ' ' -f 2
```

- 删除代码数量列

```shell
cat /home/commits.csv | cut -d '|' -f 4 | cut -d ',' -f 4 | cut -d ' ' -f 2
```



## 2 通过commit id查看对应的PR



## 2.1 修改git配置



将以下配置增加到 ~/.gitconfig中

```shell
[alias]
    find-merge = "!sh -c 'commit=$0 && branch=${1:-HEAD} && (git rev-list $commit..$branch --ancestry-path | cat -n; git rev-list $commit..$branch --first-parent | cat -n) | sort -k2 -s | uniq -f1 -d | sort -n | tail -1 | cut -f2'"
    show-merge = "!sh -c 'merge=$(git find-merge $0 $1) && [ -n \"$merge\" ] && git show $merge'"
```
然后就可以通过以下命令来查看PR的信息了
```shell
# current branch
git find-merge <SHA-1>
# specify master
git find-merge <SHA-1> master
```
通过`git show-merge`可以看到这个commit更详细的信息

