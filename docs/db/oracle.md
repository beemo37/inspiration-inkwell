<!-- omit from toc -->
# Oracle

- [通过Docker启动Oracle容器](#通过docker启动oracle容器)

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