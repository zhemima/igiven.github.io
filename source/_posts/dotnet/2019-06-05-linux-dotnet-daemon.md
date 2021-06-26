---
title:  "Linux部署dotnetcore记录"
---
## Daemon
Linux Daemon（守护进程）是运行在后台的一种特殊进程。它独立于控制终端并且周期性地执行某种任务或等待处理某些发生的事件。它不需要用户输入就能运行而且提供某种服务，不是对整个系统就是对某个用户程序提供服务。Linux系统的大多数服务器就是通过守护进程实现的。常见的守护进程包括系统日志进程syslogd、 web服务器httpd、邮件服务器sendmail和数据库服务器mysqld等...

### 配置文件

```
sudo vi /etc/systemd/system/KestrelDemoSer.service
```
修改配置
```
[Unit]
Description=KestrelDemo running on CentOS
[Service]
WorkingDirectory=/cusD/wwwroot/KesPublish
Type=simple
User=root
Group=root
ExecStart=/usr/bin/dotnet /cusD/wwwroot/KesPublish/KestrelDemo.dll
Restart=always
# Restart service after 10 seconds if the dotnet service crashes:
RestartSec=10
SyslogIdentifier=dotnet-example
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false
[Install]
WantedBy=multi-user.target

```

### 命令
```
systemctl enable KestrelDemoSer.service
systemctl start KestrelDemoSer.service
systemctl status KestrelDemoSer.service
sudo journalctl -fu KestrelDemoSer.service  //查看日志
```

## jenkins
```shell
sudo systemctl stop edu
sudo dotnet publish -c Release ${WORKSPACE}/src/Edu.Web/Edu.Web.csproj -o /usr/local/src/edu
sudo systemctl start edu
```

### efcore update

```shell
dotnet ef database update --context Edu.EntityFrameworkCore.EduMigrationsDbContext --startup-project ./src/Edu.Web/Edu.Web.csproj --project ./src/Edu.EntityFrameworkCore.DbMigrations/Edu.EntityFrameworkCore.DbMigrations.csproj   -v
```

不能使用sudo的解决办法

```
sudo visudo

jenkins ALL=(ALL) NOPASSWD: ALL

systemctl restart jenkins

```

## mysql 
从windows迁移到linux时..mysql数据库的表明可能会有大小写敏感的问题
修改mysql配置my.ini

```
lower_case_table_names = 0 //mysql会根据表名直接操作，大小写敏感。 
lower_case_table_names = 1 //mysql会先把表名转为小写，再执行操作。 
```



# windows下绝对路径启动问题

注册成services后.他的启动目录是C:\Windows\System32..所以需要手动设置下contentRoot

```
F:\DotHass.Blog\aspnet-core\src\DotHass.Blog.Web\bin\Release\publish\DotHass.Blog.Web.exe --contentRoot F:\DotHass.Blog\aspnet-core\src\DotHass.Blog.Web\bin\Release\publish
```

如果启动失败..先查看错误日志..最好是用文件存储

