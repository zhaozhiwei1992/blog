---
title: vim系列-文本操作篇
date: 2024-08-29
updated: 2024-08-29
tags: [vim, 文本]
---

# Vim实用技巧：文本编辑与处理

## 基数行与偶数行分组

使用Vim的替换命令，可以轻松地将基数行和偶数行分组：

```vim
%s/\(^.*$\)\n\(^.*$\)/\1 \2/g
```
然后，删除所有的基数行：

```vim
%s/^.*$\n\(^.*$\)/\1/g
```

## 删除重复行

在Vim中删除重复行是一个常见的操作，以下是几种方法：

### 删除相邻重复行

```vim
:g/\(.\+\)$\n\1/d
```

### 删除不相邻重复行

使用排序命令删除不相邻的重复行：

```vim
:sort u
```

### 删除重复行，结果按照原顺序排列

为了保存原有顺序，首先给每行加上行号和1个`{`：

```vim
:let i=1|g/^/s//\=i.'{'/|let i+=1
```

按照行号后面的内容排序：

```vim
:sort /^\d\{-}/
```

删除行号后面的内容相同的行保留后面的行：

```vim
:g/^\d\{-}\{.∗$\n\d\{-}\{1$/d
```

按照行号恢复顺序：

```vim
:sort n
```

## 删除空白行

删除文件中的空白行：

```vim
%g/^\s*$/d
```

## 添加序号

有多种方法可以为文本添加序号：

### 通过let变量

```vim
let i=1 | g/^/s//\=i.' '/ | let i=i+1
```

### 直接使用行号

```vim
:g/^/ s//\=line('.').' '
```

### 使用range()函数

```vim
:for i in range(1, 31)
:  call setline(i, i .' '. getline(i))
:endfor
```

### 利用Vim的编程支持

```vim
:python <<EOF
from vim import current
for i in range(len(current.buffer)):
    current.buffer[i] = str(i+1) + ' ' + current.buffer[i]
EOF
```

### 外部命令

使用外部命令如`findstr`, `sed`, `diff`, `perl`, `python`等为文本添加序号：

```vim
:%!findstr /N "^"
:%!sed =|sed "N;s/\n/ /"
:%!diff --line-format=%dn%L % -
:%!perl -pe "print ++$a . ' '"
:%!python -c "import sys,fileinput as f;[sys.stdout.write(str(f.lineno())+a) for a in f.input()]"
```

## 查看文件行数、列数、字符数及所占字节大小

查看文件的行数、列数、字符数及所占字节大小：

```vim
g + <Ctrl-g>
```
