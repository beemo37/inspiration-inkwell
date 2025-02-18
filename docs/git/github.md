<!-- omit from toc -->
# GitHub


- [搜索技巧](#搜索技巧)
    - [issue](#issue)
    - [code](#code)
      - [boolean操作符](#boolean操作符)
      - [精准字符串搜索](#精准字符串搜索)
      - [限定符](#限定符)

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

#### boolean操作符

可以使用boolean操作符来将搜索词结合起来，支持的有：

- AND：并且，空格相当于AND，如`Hello World`相当于`Hello And World`，意味着搜索结果必须包含`Hello`以及`World`。
- OR: 或者，`Hello OR World`，意味着搜索结果可以包含`Hello`或者`World`。
- NOT：不包含，否定的意思，如想搜索不在test路径但是包含Hello的文件，写法如下：`Hello NOT path:test`

#### 精准字符串搜索

当想精确搜索字符串时，可以使用引号将其包围，如：

```bash
"Hello World"
```

此种写法不会将`Hello World`看做`Hello And World`。

#### 限定符

可以通过限定符来缩小查询范围。

- 代码仓库的限定符

  关键字：`repo:`。必须指定完全的仓库路径，包括代码拥有者，如：`repo:/spring-projects/spring-framework`。如果要查询多个，使用`OR`进行连接，如：
  `repo:spring-projects/spring-framework OR repo:spring-projects/spring-boot`

- 组织限定符

  关键字：`org`，用于查询指定组织的仓库。如：`org:spring-projects`。

- 用户限定符
 
  关键字：`user`，用户查询指定用户的仓库。如：`user:xxx`。

- 语言限定符

  关键字：`language`，用于查询指定编程语言的仓库。如：`language:java`。

- 路径限定符

  关键字：`path`，用于查询指定路径文件，无论这个path是出现在路径中还是在文件名中都会匹配。如搜索`path:unit_tests`，则会匹配`/demo/unit-tests/demo.java`以及`/demo/hello/unit-tests.java`。

  可以在path中指定通配符，如`path:*.txt`，意味着搜索所有扩展名为txt的文件；`path:a?b`以为这匹配`abc`或`aac`等。

  
