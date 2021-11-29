# winexe远程调用Start-Process阻塞问题排查

## 问题描述

环境：Centos7.5，winexe1.1，windows server2008

```shell
winexe --user='admin/1234' '//10.10.10.10' "powershell (Start-Process C:\test.bat -ArgumentList 'test' -PassThru -Wait).ExitCode"
```

执行上述命令后shell无输出，敲击回车后显示输出；

## 问题解决

```shell
winexe --user='admin/1234' '//10.10.10.10' "cmd /c echo . | powershell (Start-Process C:\test.bat -ArgumentList 'test' -PassThru -Wait).ExitCode"
```

[参考一: [Why does PsExec hang after successfully running a powershell script?](https://serverfault.com/questions/437504/why-does-psexec-hang-after-successfully-running-a-powershell-script)

[参考二: Using PowerShell and PsExec to invoke expressions on remote computers](https://www.leeholmes.com/blog/2007/10/02/using-powershell-and-psexec-to-invoke-expressions-on-remote-computers/)

