<!-- omit from toc -->
# GitHub


- [搜索技巧](#搜索技巧)
    - [issue](#issue)
    - [code](#code)
      - [boolean符](#boolean符)

# 搜索技巧

### issue

### code

搜索查询由两部分组成：

- 搜索词：想要搜索的内容
- 限定符：用来缩小查询范围

当仅仅有搜索词时，会匹配文件内容或文件路径，如:
```text
hello
```

会匹配诸如`hello.txt`的文件，即使这个文件内容并不包含`hello`，也会匹配文件内容中包含`hello`的文件，即使它的名称可能叫`demo.txt`。

你也可以指定多个搜索词，使用<span style="color:red">**空格**</span>分隔，意味着需要<span style="color:red">**同时满足所有的搜索词，忽略顺序**</span>，如：

```text
spring redis
```

会匹配以下项目：
- 文件路径中包含`spring`以及`redis`，如文件路径为`RedisInSpring.txt`。
- 文件路径中包含`spring`，文件内容包含`redis`，如文件路径为`SpringDemo.txt`，文件内容包含`redis`。
- 文件路径中包含`redis`，文件内容包含`spring`，如文件路径为`RedisDemo.txt`，文件内容包含`spring`。
- 文件内容包含`spring`以及`redis`。

#### boolean符

- AND：并且，空格就相当于AND，如`Hello World`相当于`Hello And World`
- OR: 或者，`Hello OR World`
- NOT：




