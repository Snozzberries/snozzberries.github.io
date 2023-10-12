# PowerShell Remoting over SSH

When working in public cloud environments like AWS or Azure, frequently you will need to create a lot of new Windows Server instances that are short lived. Often using RDP to access these is cumbersome. You may find yourself having similar needs or may just be looking for a shell-first administration experience.

[PowerShell Remoting](https://learn.microsoft.com/en-us/powershell/scripting/learn/remoting/running-remote-commands) offers a great answer. You can simply run `Enter-PSSession` and be on your way.

The only downside is the default configuration isn't the most accommodating. The default configuration relies on WinRM, Microsoft's WS-Management protocol implementation, which provides many excellent features but requires more dependencies. You'd want to be domain joined and have proper certificates configured to really use it smoothly. More on that to come later.

You want to have a shell-first experience, but you want to use a transport protocol you can set up quickly and without many dependencies.

**Enter PowerShell Remoting over SSH!**

You will want to keep reading to see a quick way to get started!

## Prerequisites

You need to tackle a couple of prerequisites:

* Windows Server 2019 or newer
* PowerShell 7 or newer
* Local Admin Account

## Installing OpenSSH Server

You will use the below combo to get the latest version of OpenSSH Server and add the Windows Capability Package to your instance.

The `Get-WindowsCapability -Online` command provides you with all the capability packages available to you.

You pass all those packages through the pipeline to `|?{$_.Name -like "OpenSSH.Server*"}`. Using the `?{}` alias for `Where-Object` and you target the `Name` property of your capability packages that begins with "OpenSSH.Server".

Finally, you pass the capability packages that match your property criteria through the pipeline to `|Add-WindowsCapability -Online`. This will acquire and install the capability package on your instance.

```powershell
Get-WindowsCapability -Online|?{$_.Name -like "OpenSSH.Server*"}|Add-WindowsCapability -Online
```

## Testing OpenSSH Server

Now that you installed the OpenSSH Server. You will want to start it and make sure it is able to run.

Use `Start-Service sshd` to turn on the server and then `Test-NetConnection localhost -Port 22` to see if the server is listening on the default SSH port.

```powershell
Start-Service sshd
Test-NetConnection localhost -Port 22
```

> If you run into challenges here. You could have other processes bound to port 22 already, you may have older versions of OpenSSH Server installed, or a number of other issues. The Windows Event Log would hopefully provide a good starting point for what may be an issue.

## Configure Password Authentication

Since you want a simple path forward, you will just enable password authentication. 

With the commands below you use the `cp` alias to copy the current OpenSSH Server config and append `.old` to the filename. Then you use the `gc` alias to get the contents of the config file with `-repace` to uncomment the specific config line. You finish by passing the content with the change through the pipeline to the `sc` alias to set the content for the config file. Finally, you can restart the service to have the new config file take effect.

```powershell
cp "$env:ProgramData\ssh\sshd_config" "$env:ProgramData\ssh\sshd_config.old"
(gc "$env:ProgramData\ssh\sshd_config") -replace "#PasswordAuthentication yes", "PasswordAuthentication yes" | sc $env:ProgramData\ssh\sshd_config
Restart-Service sshd
```

## Connecting to Your Server

You can now use [Putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html#:~:text=putty.exe) or another SSH client. When you connect, your session prompt will default to Windows Command Prompt (i.e., `cmd.exe`).

> You will not be able to use PowerShell Remoting with `Enter-PsSession` at this point. 

You will set the default prompt to PowerShell Core using the `New-ItemPropert` commandlet to set the `DefaultShell` value of the `OpenSSH` key within the `HKEY_LOCAL_MACHINE\SOFTWARE` hive registry path. The `data` within the key's value is the path to the PowerShell Core executable using the short filename path (i.e., 8.3 filename). The value of `c:/progra~1/powershell/7/pwsh.exe` is the default location. Then you will need to restart the SSH service.

When you next connect you will see your PowerShell Core prompt.

```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "c:/progra~1/powershell/7/pwsh.exe" -PropertyType String -Force
Restart-Service sshd
```

### OpenSSH Subsystem

To enable the PowerShell Remoting functionality (e.g., `Enter-PsSession`) you will need to configure an [OpenSSH Subsystem](https://learn.microsoft.com/en-us/powershell/scripting/learn/remoting/ssh-remoting-in-powershell-core?view=powershell-7.3#overview). This will allow OpenSSH to create a PowerShell process for the remoting session to connect with.

With the commands below you use the `gc` alias to get the contents of the config file with `-repace` to uncomment the specific config line. You finish by passing the content with the change through the pipeline to the `sc` alias to set the content for the config file. You can restart the service for the new config file take effect.

```powershell
(gc "$env:ProgramData\ssh\sshd_config") -replace "# override default of no subsystems", "$&`nSubsystem powershell c:/progra~1/powershell/7/pwsh.exe -sshs -nologo -noprofile" | sc $env:ProgramData\ssh\sshd_config
Restart-Service sshd
```

## Putting It All Together

Circling back on how to apply all of this. You can use the following as an AWS User Data script to:

* Identify, download, and install the latest version of PowerShell Core.
* Acquire, install, and set the startup to automatic for OpenSSH Server service.
* Attempt to add a firewall rule allowing access to OpenSSH Server.
* Setting the default shell to PowerShell Core for OpenSSH Server interactive sessions.
* Create a backup copy of the default `sshd_config` file.
* Configure PowerShell Core as an OpenSSH Server Subsystem to allow for asynchronous commands.
* Configure OpenSSH Server to allow password authentication.
* Apply the configuration changes with a restart of the OpenSSH Server service.

```powershell
<powershell>
$r=[System.Net.WebRequest]::Create("https://github.com/PowerShell/PowerShell/releases/latest")
$r.AllowAutoRedirect=$false
$r.Method="Head"
$t=$r.GetResponse().Headers["Location"]
$v=$t.substring($t.indexOf("/tag/v")+6,$t.length-$t.indexOf("/tag/v")-6)
irm https://github.com/PowerShell/PowerShell/releases/download/v$v/PowerShell-$v-win-x64.msi -OutFile ".\Powershell-$v-win-x64.msi"
Start-Process msiexec.exe -Wait -ArgumentList "/I Powershell-$v-win-x64.msi /quiet /l*v .\Powershell-$v-win-x64.msi.log"
Get-WindowsCapability -Online|?{$_.Name -like "OpenSSH.Server*"}|Add-WindowsCapability -Online
Start-Service sshd
Set-Service -Name sshd -StartupType 'Automatic'
New-NetFirewallRule -Name 'OpenSSH-Server-In-TCP' -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "c:/progra~1/powershell/7/pwsh.exe" -PropertyType String -Force
cp "$env:ProgramData\ssh\sshd_config" "$env:ProgramData\ssh\sshd_config.old"
(gc "$env:ProgramData\ssh\sshd_config") -replace "# override default of no subsystems", "$&`nSubsystem powershell c:/progra~1/powershell/7/pwsh.exe -sshs -nologo -noprofile" | sc $env:ProgramData\ssh\sshd_config
(gc "$env:ProgramData\ssh\sshd_config") -replace "#PasswordAuthentication yes", "PasswordAuthentication yes" | sc $env:ProgramData\ssh\sshd_config
Restart-Service sshd
</powershell>
```

## Additional Notes

With this you have just scratched the surface of using PowerShell Remoting. Here are a few other areas you could continue learning about.

* The implementation of OpenSSH Server is not full feature parity. Microsoft does [document the configuration options](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_server_configuration) for your review.
* Although password based authentication is easy, using [certificate based authentication is preferable and supported](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement).
* This is a great option if you want simple and quick, but as you begin to need more complex identity management integrations [WinRM can become more favorable](https://learn.microsoft.com/en-us/powershell/scripting/learn/remoting/wsman-remoting-in-powershell-core).
* Jordan Borean has delved deep on these topics and shares many great insights with you on [his blog](https://www.bloggingforlogging.com/).
