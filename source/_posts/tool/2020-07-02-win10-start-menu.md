---
title : "win10开始菜单添加软件以及备份和恢复" 
---



# 添加应用到开始屏幕

右键默认就有添加到开始屏幕

# 备份与恢复

默认布局位于 C:\Users\username\AppData\Local\Microsoft\Windows\Shell\ ..注意这是默认布局

真实的布局是存在与注册表中

```
Export-StartLayout –path c:\start\start.xml
Import-StartLayout -layoutpath c:\start\start.xml -mountpath "C:\"
```









## 其他路径

C:\ProgramData\Microsoft\Windows\Start Menu\Programs

C:\Users\zhepama\AppData\Roaming\Microsoft\Windows\Start Menu\Programs



## 相关链接

- [https://docs.microsoft.com/zh-cn/windows/configuration/customize-and-export-start-layout#export-the-start-layout](https://docs.microsoft.com/zh-cn/windows/configuration/customize-and-export-start-layout#export-the-start-layout)
- [https://docs.microsoft.com/en-us/powershell/module/startlayout/import-startlayout?view=win10-ps](https://docs.microsoft.com/en-us/powershell/module/startlayout/import-startlayout?view=win10-ps)

