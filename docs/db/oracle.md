<!-- omit from toc -->
# Oracle

- [通过Docker启动Oracle容器](#通过docker启动oracle容器)
- [生成随机字符串](#生成随机字符串)
- [插入时如果重复则忽略](#插入时如果重复则忽略)
- [局部索引](#局部索引)

## 通过Docker启动Oracle容器

下载好镜像后（可用gvenzl/oracle-xe:18-slim），启动容器时，需要指定SYSTEM用户的密码，否则报错
```bash
docker run -d --name my_oracle -p 1521:1521 -e ORALCE_PASSWORD=<PASSWORD> <镜像名>
```

如果使用`PLSQL Developer`则可以在`tnsnames.ora`中进行如下配置

```
# 本地库，local为别名，自定义填写
local=
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = XE)
    )
)
```

## 生成随机字符串

```sql
-- 生成32位字符串
SELECT DBMS_RANDOM.STRING('x', 32) FROM DUAL;
```

- u/U：大写字母组成的字符串
- l/L：小写字母组成的字符串
- a/A：大小写字母混合组成的字符串
- x/X：大写字母和数字组成的字符串
- p/P：任意可打印字符组成的字符串（有空格，符号等）

```sql
-- 生成全局UID
SELECT * RAWTOHEX(SYS_GUID()) FROM DUAL;
```

## 插入时如果重复则忽略

```sql
INSERT /*+ IGNORE_ROW_ON_DUPKEY_INDEX(uni_instance(type))*/ INTO uni_instance(instance_id) VALUES (LOWER(RAWTOHEX(SYS_GUID()))) ;
```

用于优化器提供执行计划的额外信息。这个特定的提示`IGNORE_ROW_ON_DUPKEY_INDEX`告诉数据库，在尝试插入重复的键值时忽略当前行，而不是抛出错误。

## 局部索引

`MySQL`以及`PostgreSQL`中，如唯一键有null则该列数据不参加唯一键的运算，但是Oracle中会把null当作普通数值进行计算，所以需要进行额外的设置局部索引

```sql
-- 如果PATH不是NULL则APP_ID、DATA_TYPE、TENANT以及PATH作为唯一键参与校验，否则不作为索引
CREATE UNIQUE INDEX UNI_DATA_PERMISSION_IDX2 ON UNI_DATA_PERMISSION(CASE WHEN PATH IS NOT NULL THEN APP_ID||'-'||DATA_TYPE||'-'||TENANT||'-'||PATH END)
```

