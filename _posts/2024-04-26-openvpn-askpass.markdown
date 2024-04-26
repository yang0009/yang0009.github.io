---
layout: post
title:  "windows 后台进程如何设置Start-job指令"
date:   2024-03-04 12:26:23 +0530
categories: dockerd
---
背景:
pycharm terminal 启动 长时间运行会导致pychram崩溃，导致服务挂掉，改进为start-job 命令发布后端服务

# 步骤
1： 开启powershell 管理员模式     
2：get-job 查看后台job     
3：启动job  
	start-job ****  
4: 停止job  
  strop-job -Id ***  
5: 重启job  
	没有重启命令；按照3，4步骤重新走一遍  

# 启动命令 
```
Start-job -Name test2  -ScriptBlock { Set-Location C:\Users\test_1\Desktop\test2; python .\project_server\server.py >> "C:\Users\test_1\Desktop\logs\test2.log"}
```
使用 -Name 参数在 Start-Job 命令中指定一个 Job 的名称。这样可以方便你在后续的操作中引用这个 Job, 在 PowerShell 的 -ScriptBlock 中，你可以使用 Set-Location 命令（或者它的别名 cd）来改变当前的工作目录; 另外 ">>" 为重定向日志符号


# 建议
这里启动可以选择启动多个shell , 在 PowerShell 中，Jobs 是与创建它们的 session（会话）相关联的。当你关闭 PowerShell session（例如，关闭 PowerShell 窗口或终端），所有在该 session 中创建的 Jobs 都会被停止并删除。如果你需要创建一个可以在关闭 PowerShell session 后继续运行的后台任务，你可能需要考虑使用 Scheduled Tasks（计划任务）或者使用 nohup（在 Unix-like 系统中）等工具。对于 Windows，你可以使用 schtasks 命令来创建一个计划任务。以下是一个简单的例子：schtasks /create /tn "MyTask" /tr "powershell -Command {Get-Process > 'C:\Path\To\Your\Log\File.txt'}" /sc once /st 00:00这个命令创建了一个名为 "MyTask" 的计划任务，该任务会在指定的时间（在这个例子中是 00:00）运行一个 PowerShell 命令（获取所有进程信息并将其保存到一个文件中）。你可以根据你的需要修改这个命令。请注意，创建计划任务可能需要管理员权限。如果你在运行上述命令时遇到问题，你可能需要以管理员身份运行 PowerShell
