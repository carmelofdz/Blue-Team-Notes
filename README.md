<img src="https://user-images.githubusercontent.com/44196051/120006585-f0dc3c00-bfd0-11eb-98d9-da3eb59edbda.png" width="250" height="250">

# Blue Team Notes
A collection of one-liners useful for blue team work. Tend to be Powershell as these are the ones I forget the most. I've generally used these one liners on [Velociraptor](https://www.velocidex.com), which can query a thousands of endpoints at once.

I use _sysmon_ and _memetask_ as file or directory names in lieu of real file names, just replace the stupid names I've given with the files you actually need. 

I've tried not to use PwSh abbrevations or alias' without first writing out a verbose version of the commands. This is so no one feels fustrated that they aren't sure what a particular command is doing if it's their first time seeing the alias.

I've included screenshots where possible so you know what you're getting. Some screenshots will be from a Win machine, others may be from the Velociraptor GUI but they do the same thing as if you were on a host's powershell command line.

## Contact me
If you see a mistake, or have an easier way to run a command then you're welcome to hit me up on [Twitter](https://twitter.com/Purp1eW0lf) or commit an issue here. 

If you want to contribute I'd be grateful for the command and a screenshot. I'll of course add you as a contributor

## Table of Contents
- [Shell Style](#shell-style)
- [Powershell](#Powershell)
  * [OSInfo](#os-info)
  * [Service Queries](#service-queries)
  * [User Queries](#user-queries)
  * [Network Queries](#network-queries)
  * [Process Queries](#process-queries)
  * [Sch Task Queries](#sch-task-queries)
  * [File Queries](#file-queries)
  * [Reg Queries](#reg-queries)
  * [Log Troubleshooting](#log-troubleshooting)
  * [Code Red](#code-red)
  * [Powershell Tips](#powershell-tips)
- [Linux](#linux)
  * [Bash History](#bash-history)
  * [Grep and Ack](#grep-and-ack)
  * [Rapid Malware Analaysis](#rapid-malware-analaysis)
  * [Processes and Networks](#processes-and-networks)
  * [Files](#files)
  * [Bash Tips](#bash-tips)

# Shell Style
### Give shell timestamp
For screenshots during IR, I like to have the date, time, and sometimes the timezone in my shell
#### CMD
```bat
setx prompt $D$S$T$H$H$H$S$B$S$P$_--$g
:: all the H's are to backspace the stupid microsecond timestamp
:: $_ and --$g seperate the date/time and path from the actual shell
:: We make the use of the prompt command: https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/prompt
:: setx is in fact the command line command to write variables to the registery
:: We are writing the prompt's new timestamp value in the cmd line into the reg so it stays, otherwise it would not stay in the cmdline when we closed it.
```
![image](https://user-images.githubusercontent.com/44196051/119978466-97b0e000-bfb1-11eb-83e1-022efba7dc96.png)

#### Pwsh
```powershell
###create a powershell profile, if it doesnt exist already
New-Item $Profile -ItemType file –Force
##open it in notepad to edit
function prompt{ "[$(Get-Date)]" +" | PS "+ "$(Get-Location) > "}
##risky move, need to tighten this up. Change your execution policy or it won't
#run the profile ps1
#run as powershell admin
Set-ExecutionPolicy RemoteSigned
```
![image](https://user-images.githubusercontent.com/44196051/119978226-486aaf80-bfb1-11eb-8e9e-52eabf2cde4c.png)

#### Bash
```bash
##open .bashrc
sudo nano .bashrc
#https://www.howtogeek.com/307701/how-to-customize-and-colorize-your-bash-prompt/
##date, time, colour, and parent+child directory only, and -> promptt
PS1='\[\033[00;35m\][`date  +"%d-%b-%y %T %Z"]` ${PWD#"${PWD%/*/*}/"}\n\[\033[01;36m\]-> \[\033[00;37m\]'
      ##begin purple  #year,month,day,time,timezone #show last 2 dir #next line, cyan,->prompt #back to normal white text
#restart the bash source
source ~/.bashrc
```
![image](https://user-images.githubusercontent.com/44196051/119981537-a7cabe80-bfb5-11eb-8b7e-1e5ba7f5ba99.png)

# Powershell
## OS Info
### Get OS and Pwsh info
This will print out the hostname, the OS build info, and the powershell version
```powershell
$Bit = (get-wmiobject Win32_OperatingSystem).OSArchitecture ; 
$V = $host | select-object -property "Version" ; 
$Build = (Get-WmiObject -class Win32_OperatingSystem).Caption ; 
write-host "$env:computername is a $Bit $Build with Pwsh $V
```
![image](https://user-images.githubusercontent.com/44196051/120313634-2be0b700-c2d2-11eb-919f-5792169a1dba.png)

#### Get Fully Qualified Domain Name
```powershell
([System.Net.Dns]::GetHostByName(($env:computerName))).Hostname
```

### Time info
#### Human Readable
Get something human readable
```powershell
Get-Date -UFormat "%a %Y-%b-%d %T UTC:%Z" 
```
![image](https://user-images.githubusercontent.com/44196051/120298372-f03df100-c2c1-11eb-92ab-d642c26133ab.png)

#### Machine comparable
This one is great for doing comparisons
```powershell
[Xml.XmlConvert]::ToString((Get-Date).ToUniversalTime(), [System.Xml.XmlDateTimeSerializationMode]::Utc) 
```
![image](https://user-images.githubusercontent.com/44196051/120314399-1e77fc80-c2d3-11eb-9a75-f9e677153d86.png)

#### Compare UTC time from Local time
```powershell
$Local = get-date;$UTC = (get-date).ToUniversalTime();
write-host "LocalTime is: $Local";write-host "UTC is: $UTC"
```
![image](https://user-images.githubusercontent.com/44196051/120301782-1fa22d00-c2c5-11eb-908f-763897fac25f.png)

### Update Info

#### Get Patches
Will show all patch IDs and their installation date
```powershell
get-hotfix|
select-object HotFixID,InstalledOn|
Sort-Object  -Descending -property InstalledOn|
format-table -autosize
```
![image](https://user-images.githubusercontent.com/44196051/120307390-d5bc4580-c2ca-11eb-8ffe-d1a835b1ce40.png)

#### Manually check if patch has taken
This happened to me during the March 2021 situation with Microsoft Exchange's ProxyLogon. The sysadmin swore blind they had patched the server, but neither `systeminfo` of `get-hotfix` was returning with the correct KB patch.

The manual workaround isn't too much ballache

##### Microsoft Support Page
First identify the ID number of the patch you want. And then find the dedicated Microsoft support page for it. 

For demonstration purposes, let's take `KB5001078` and it's [corresponding support page](https://support.microsoft.com/en-us/topic/kb5001078-servicing-stack-update-for-windows-10-version-1607-february-12-2021-3e19bfd1-7711-48a8-978b-ce3620ec6362). You'll be fine just googling the patch ID number.

![image](https://user-images.githubusercontent.com/44196051/120308871-7a8b5280-c2cc-11eb-850f-da46727a94ac.png)

Then click into the dropdown relevant to your machine. 
![image](https://user-images.githubusercontent.com/44196051/120309734-7f043b00-c2cd-11eb-9cf6-3a7ca6be6691.png)

Here you can see the files that are included in a particular update. The task now is to pick a handful of the patch-files and compare your host machine. See if these files exist too, and if they do do they have similar / same dates on the host as they do in the Microsoft patch list?

##### On Host
Let us now assume you don't know the path to this file on your host machine. You will have to recursively search for the file location. It's a fair bet that the file will be in `C:\Windows\` (but not always), so lets' recursively look for `EventsInstaller.dll`

```powershell
$file = 'EventsInstaller.dll'; $directory = 'C:\windows' ;
gci -Path $directory -Filter $file -Recurse -force|
sort-object  -descending -property LastWriteTimeUtc | fl *
```
We'll get a lot of information here, but we're really concerned with is the section around the various *times*. As we sort by the `LastWriteTimeUtc`, the top result should in theory be the latest file of that name...but this is not always true.

![image](https://user-images.githubusercontent.com/44196051/120312109-37cb7980-c2d0-11eb-95e2-8655cd89f9cc.png)

##### Discrepencies
I've noticed that sometimes there is a couple days discrepency between dates. 

![image](https://user-images.githubusercontent.com/44196051/120313127-7d3c7680-c2d1-11eb-8941-e96575a63138.png)

For example in our screenshot, on the left Microsoft's support page supposes the `EvenntsInstaller.dll` was written on the 13th January 2021. And yet our host on the right side of the screenshot comes up as the 14th January 2021. This is fine though, you've got that file don't sweat it. 

## Account Queries
### Users recently created in Active Directory
*Run on a Domain Controller*.

Change the AddDays field to more or less days if you want. Right now set to seven days.

The 'when Created' field is great for noticing some inconsistencies. For example, how often are users created at 2am?
```powershell
import-module ActiveDirectory;
$When = ((Get-Date).AddDays(-7)).Date; Get-ADUser -Filter {whenCreated -ge $When} -Properties whenCreated
```
![image](https://user-images.githubusercontent.com/44196051/120461945-614cd980-c392-11eb-8352-2141ee42efdf.png)

### Hone in on suspicious user
You can use the `SamAccountName` above to filter
```powershell
import-module ActiveDirectory;
Get-ADUser -Identity HamBurglar -Properties *
```
![image](https://user-images.githubusercontent.com/44196051/120328655-f1334a80-c2e2-11eb-97da-653553b7c01a.png)

### Retrieve local user accounts that are enabled
```powershell
 Get-LocalUser | ? Enabled -eq "True"
```
![image](https://user-images.githubusercontent.com/44196051/120561793-216f0c00-c3fd-11eb-9738-76e778c79763.png)

### Find all users currently logged in
```powershell
Get-CimInstance -classname win32_computersystem |
select username, domain, DNSHostName | ft -autosize
```
![image](https://user-images.githubusercontent.com/44196051/120562311-1072ca80-c3fe-11eb-995f-9d42d1c451d6.png)


### Computer / Machine Accounts
Adversaries like to use Machine accounts (accounts that have a $) as these often are overpowered AND fly under the defenders' radar

#### Show machine accounts that are apart of interesting groups. 
There may be misconfigurations that an adversary could take advantadge. 
```powershell
Get-ADComputer -Filter * -Properties MemberOf | ? {$_.MemberOf}
```
![image](https://user-images.githubusercontent.com/44196051/120346984-cac9db00-c2f3-11eb-8ab0-1112aa2183a9.png)

#### Reset password for a machine account. 
Good for depriving adversary of pass they may have got. 
Also good for re-establishing trust if machine is kicked out of domain trust for reasons(?)

```powershell
Reset-ComputerMachinePassword
```

## Service Queries

### Show Services & Service Accounts
Utilise Get-WmiObject(gwmi) to show all service accounts on a machine, and then sort to show the running accounts first and the stopped accounts second.

StartName is the name of the Service Account btw

```powershell
 gwmi -Class Win32_Service|
 select-object -Property Name, StartName, state, startmode, Caption, ProcessId |
 sort-object -property state
```
![image](https://user-images.githubusercontent.com/44196051/120340649-23967500-c2ee-11eb-892b-0c6626072d8c.png)

### Hone in on specific Service
If a specific service catches your eye, you can get all the info for it.  Because the single and double qoutes are important to getting this right, I find it easier to just put the DisplayName of the service I want as a variable, as I tend to fuck up the displayname filter bit

```powershell
$DisName = "Active Directory Web Services"; 
gwmi -Class Win32_Service -Filter "Displayname = '$DisName' " | fl *
```
![image](https://user-images.githubusercontent.com/44196051/120341774-14fc8d80-c2ef-11eb-8b1d-31db7620b7cb.png)

### Kill a service
``` powershell
Get-Service -DisplayName "meme_service" | Stop-Service -force -confirm
```

## Network Queries
### Find internet established connections, and sort by time established
You can always sort by whatever value you want really. CreationTime is just an example
```powershell
Get-NetTCPConnection -AppliedSetting Internet |
select-object -property remoteaddress, remoteport, creationtime |
Sort-Object -Property creationtime |
format-table -autosize
```
![image](https://user-images.githubusercontent.com/44196051/120314725-73b40e00-c2d3-11eb-9dbf-3b0582a9b2d0.png)

### Sort remote IP connections, and then unique them
This really makes strange IPs stand out
```powershell
(Get-NetTCPConnection).remoteaddress | Sort-Object -Unique 
```
![image](https://user-images.githubusercontent.com/44196051/120314835-8dedec00-c2d3-11eb-8469-e658eb743364.png)

#### Hone in on a suspicious IP
If you see suspicious IP address in any of the above, then I would hone in on it
```powershell
Get-NetTCPConnection |
? {($_.RemoteAddress -eq "1.2.3.4")} |
select-object -property state, creationtime, localport,remoteport | ft -autosize

## can do this as well 
 Get-NetTCPConnection -remoteaddress 0.0.0.0 |
 select state, creationtime, localport,remoteport | ft -autosize
 ```
 ![image](https://user-images.githubusercontent.com/44196051/120313809-68141780-c2d2-11eb-85ac-5e369715f8ed.png)

### Show UDP connections
You can generally filter pwsh UDP the way we did the above TCP
```powershell
 Get-NetUDPEndpoint | select local*,creationtime, remote* | ft -autosize
```
![image](https://user-images.githubusercontent.com/44196051/120562989-7744b380-c3ff-11eb-963b-443cc9176643.png)


### Kill a connection
There's probably a better way to do this. But essentially, get the tcp connection that has the specific remote IPv4/6 you want to kill. It will collect the OwningProcess. From here, get-process then filters for those owningprocess ID numbers. And then it will stop said process. Bit clunky
``` powershell
stop-process -confirm (Get-Process -Id (Get-NetTCPConnection -RemoteAddress "1.2.3.4" ).OwningProcess)
```

### Firewall Queries
Should this get it's own section? I _feel_ like it's part of network queries

#### Retreieve Firewall profile names
```powershell
(Get-NetFirewallProfile).name
```
![image](https://user-images.githubusercontent.com/44196051/120560271-53cb3a00-c3fa-11eb-83f7-f60f431d0c7b.png)

#### Retrieve rules of specific profile
Not likely to be too useful getting all of this information raw, so add plenty of filters
```powershell
Get-NetFirewallProfile -Name Public | Get-NetFirewallRule
##filtering it to only show rules that are actually enabled
Get-NetFirewallProfile -Name Public | Get-NetFirewallRule | ? Enabled -eq "true"
```
![image](https://user-images.githubusercontent.com/44196051/120560766-3cd91780-c3fb-11eb-9781-bf933c4b0efa.png)

#### Filter all firewall rules
```powershell

#show firewall rules that are enabled
Get-NetFirewallRule | ? Enabled -eq "true"
#will show rules that are not enabled
Get-NetFirewallRule | ? Enabled -notmatch "true"

##show firewall rules that pertain to inbound
Get-NetFirewallRule | ? direction -eq "inbound"
#or outbound
Get-NetFirewallRule | ? direction -eq "outbound"

##stack these filters
Get-NetFirewallRule | where {($_.Enabled -eq "true" -and $_.Direction -eq "inbound")}
#or just use the built in flags lol
Get-NetFirewallRule -Enabled True -Direction Inbound
```

## Process Queries

#### Processes and TCP COnnections
Collect the owningprocess of the TCP connections, and then ask get-process to filter and show processes that make network communications

```powershell
Get-Process -Id (Get-NetTCPConnection).OwningProcess
```
![image](https://user-images.githubusercontent.com/44196051/120337318-1cba3300-c2eb-11eb-8444-0b54e67f6285.png)

### Show all processes and their associated user
```powershell
get-process * -Includeusername
```

![image](https://user-images.githubusercontent.com/44196051/120329122-70288300-c2e3-11eb-95ef-276ffd556acd.png)

### Get specific info about the full path binary that a process is running
```powershell
get-process -name "memetask" | select-object -property Name, Id,Includeusername, Path
```
![image](https://user-images.githubusercontent.com/44196051/119979341-bb285a80-bfb2-11eb-89a8-83b4c8f732c5.png)

### Is a specific process a running on a machine or not
```powershell
$process = "memes";
if (ps |  where-object ProcessName -Match "$process") {Write-Host "$process successfully installed on " -NoNewline ; hostname} else {write-host "$process absent from " -NoNewline ; hostname}
```

Example of process that is absent
![image](https://user-images.githubusercontent.com/44196051/119976215-b1045d00-bfae-11eb-806c-49a62f5aab15.png)
Example of process that is present
![image](https://user-images.githubusercontent.com/44196051/119976374-ea3ccd00-bfae-11eb-94cd-37ed4233564d.png)

### Get process hash
Great to make malicious process stand out. If you want a different Algorithm, just change it after `-Algorithm` to something like `sha256` 
```powershell
foreach ($proc in Get-Process | select path -Unique){try
{ Get-FileHash $proc.path -Algorithm md5 -ErrorAction stop|
Select-Object -property hash,path}catch{}}
```
![image](https://user-images.githubusercontent.com/44196051/119976802-8cf54b80-bfaf-11eb-82de-1a92bbcae4f9.png)

### Show all DLLs loaded with a process
```powershell
get-process -name "memestask" -module 
```
![image](https://user-images.githubusercontent.com/44196051/119976958-bdd58080-bfaf-11eb-8833-7fdf78045967.png)

Alternatively, pipe `|fl` and it will give a granularity to the DLLs

![image](https://user-images.githubusercontent.com/44196051/119977057-db0a4f00-bfaf-11eb-97ce-1e762088de8e.png)

### Identify process CPU usage 
```powershell
 (Get-Process -name "googleupdate").CPU | fl 
```
![image](https://user-images.githubusercontent.com/44196051/119982198-756d9100-bfb6-11eb-8645-e41cf46116b3.png)

I get mixed results with this command but it's supposed to give the percent of CPU usage.
```powershell
$ProcessName = "symon" ; 
$ProcessName = (Get-Process -Id $ProcessPID).Name; 
$CpuCores = (Get-WMIObject Win32_ComputerSystem).NumberOfLogicalProcessors; 
$Samples = (Get-Counter "\Process($Processname*)\% Processor Time").CounterSamples; 
$Samples | Select `InstanceName,@{Name="CPU %";Expression={[Decimal]::Round(($_.CookedValue / $CpuCores), 2)}}
```
![image](https://user-images.githubusercontent.com/44196051/119982326-9a620400-bfb6-11eb-9a66-ad5a5661bc8a.png)

### Sort by least CPU-intensive processes
On the second line, you can change the `-last` flag to `-first`. Right now will show the lower cpu-using proccesses...useful as malicious process probably won't be as big a CPU as Chrome, for example. But change to `first` if you want to see the chungus processes
```powershell
Get-Process | Sort CPU -descending |
Select -last 20 -Property ID,ProcessName,CPU 
| format-table -autosize
```
![image](https://user-images.githubusercontent.com/44196051/120009189-9db7b880-bfd3-11eb-860f-8de8446550eb.png)

### Stop a Process
```powershell
Stop-Process -Name "memeprocess"
### if persists
Get-Process -Name "memeprocess" | Stop-Process -Force
```

## Sch Task Queries

### Get a specific schtask
```powershell
Get-ScheduledTask -Taskname "wifi*"

## or just list them all
Get-ScheduledTask
```
![image](https://user-images.githubusercontent.com/44196051/120563312-2d100200-c400-11eb-8f47-cd3e76df4165.png)

### To find the commands a task is running
Identify the user behind a command. Great at catching out malicious schtasks that perhaps are imitating names, or a process name
```powershell
Get-ScheduledTask | Select-Object -Property TaskName,author | fl 
```
![image](https://user-images.githubusercontent.com/44196051/119978821-01c98500-bfb2-11eb-9149-fc1a96a1af87.png)

Great one liner to find exactly WHAT a regular task is doing
```powershell
$task = Get-ScheduledTask | where TaskName -EQ "meme task"; 
$task.Actions
```
![image](https://user-images.githubusercontent.com/44196051/119979087-5f5dd180-bfb2-11eb-9d4d-bbbf66043535.png)

And a command to get granularity behind the schtask requires you to give the taskpath. Tasks with more than one taskpath will throw an error here
```powershelll
$task = "CacheTask";
get-scheduledtask -taskpath (Get-ScheduledTask -Taskname "$task").taskpath | Export-ScheduledTask
#this isn't the way the microsoft docs advise. 
     ##But I prefer this, as it means I don't need to go and get the taskpath when I already know the taskname
```
![image](https://user-images.githubusercontent.com/44196051/120563667-18803980-c401-11eb-9b10-621169f38437.png)


### To stop the task
```powershell
Get-ScheduledTask "memetask" | Stop-ScheduledTask
```
### Show what programs run at startup
```powershell
Get-CimInstance Win32_StartupCommand | Select-Object Name, command, Location, User | Format-List 
```
![image](https://user-images.githubusercontent.com/44196051/120332890-12963580-c2e7-11eb-9805-feee341140fa.png)




## File Queries
### Check if a specific file or path is alive. 

I've found that this is a great one to quickly check for specific vulnerabilities. Take for example, CVE-2021-21551. The one below this one is an excellent way of utilising the 'true/false' binary results that test-path can give
``` powershell
test-path -path "C:\windows\temp\DBUtil_2_3.Sys"
```
![image](https://user-images.githubusercontent.com/44196051/119982761-283def00-bfb7-11eb-83ab-061b6c628372.png)

### test if  files and directories are present or absent
This is great to just sanity check if things exist. Great when you're trying to check if files or directories have been left behind when you're cleaning stuff up.
```powershell
$a = Test-Path "C:\windows\sysmon.exe"; $b= Test-Path "C:\Windows\SysmonDrv.sys"; $c = test-path "C:\Program Files (x86)\sysmon"; $d = test-path "C:\Program Files\sysmon"; 
IF ($a -eq 'True') {Write-Host "C:\windows\sysmon.exe present"} ELSE {Write-Host "C:\windows\sysmon.exe absent"}; 
IF ($b -eq 'True') {Write-Host "C:\Windows\SysmonDrv.sys present"} ELSE {Write-Host "C:\Windows\SysmonDrv.sys absent"} ; 
IF ($c -eq 'True') {Write-Host "C:\Program Files (x86)\sysmon present"} ELSE {Write-Host "C:\Program Files (x86)\sysmon absent"}; 
IF ($d -eq 'True') {Write-Host "C:\Program Files\sysmon present"} ELSE {Write-Host "C:\Program Files\sysmon absent"}
```
![image](https://user-images.githubusercontent.com/44196051/119979754-443f9180-bfb3-11eb-9259-5409a0d98c04.png)

^ The above is a bit over-engineered. Here's an an abbrevated version
```powershell
$Paths = "C:\windows" , "C:\temp", "C:\windows\system32", "C:\DinosaurFakeDir" ; 
foreach ($Item in $Paths){if
(test-path $Item) {write "$Item present"}else{write "$Item absent"}}
```
![image](https://user-images.githubusercontent.com/44196051/120552156-c7ffe080-c3ee-11eb-8f81-3983cab8083b.png)

We can also make this conditional. Let's say if Process MemeProcess is NOT running, we can then else it to go and check if files exist
```powershell
$Paths = "C:\windows" , "C:\temp", "C:\windows\system32", "C:\DinosaurFakeDir" ; 
if (Get-Process | where-object Processname -eq "explorer") {write "process working"} else {
foreach ($Item in $Paths){if (test-path $Item) {write "$Item present"}else{write "$Item absent"}}}
```
![image](https://user-images.githubusercontent.com/44196051/120553995-1c0bc480-c3f1-11eb-811d-eca65d10328d.png)

You can use `test-path` to query Registry, but even the 2007 [Microsoft docs say](https://devblogs.microsoft.com/powershell/test-path-we-goofed/) that this can give inconsistent results, so I wouldn't bother with test-path for reg stuff when it's during an IR

### Get File info
Seen a file you don't recognise? Find out some more about it! Remember though: don't trust timestamps!
```powershell
Get-item C:\Temp\Computers.csv |
select-object -property @{N='Owner';E={$_.GetAccessControl().Owner}}, *time, versioninfo | fl 
```
![image](https://user-images.githubusercontent.com/44196051/120334042-3443ec80-c2e8-11eb-84a9-c141ca5198a8.png)

### Recursively look for particular file types, and once you find the files get their hashes
This one-liner was a godsend during the Microsoft Exchange ballache back in early 2021

You may find that the fullpath gets cut off with elpises (...). To counter this, pipe to ` format-table -wrap`. It will look ugly, but will ensure your stuff doesn't get cut off. 
```powershell
Get-ChildItem -path "C:\windows\temp" -Recurse -Force -File -Include *.aspx, *.js, *.zip|
Get-FileHash |
Select-Object -property hash, path
#optionally | format-table -wrap
```
![image](https://user-images.githubusercontent.com/44196051/119977578-887d6280-bfb0-11eb-9e56-fad64296128f.png)


### Find files written after X date
I personally wouldn't use this for DFIR. It's easy to manipulate timestamps....plus, Windows imports the original compiled date for some files and binaries if I'm not mistaken

Change the variables in the first time to get what you're looking
```powershell
$date = "12/01/2021"; $directory = "C:\temp"
get-childitem "$directory" -recurse|
where-object {$_.mode -notmatch "d"}| 
where-object {$_.lastwritetime -gt [datetime]::parse("$date")}|
Sort-Object -property LastWriteTime | format-table lastwritetime, fullname -autosize
```

![image](https://user-images.githubusercontent.com/44196051/120306808-2b442280-c2ca-11eb-82f8-bca23b5ee0d1.png)

### copy multiple files to new location
```powershell
copy-item "C:\windows\System32\winevt\Logs\Security.evtx", "C:\windows\System32\winevt\Logs\Windows PowerShell.evtx" -destination C:\temp
```

## Reg Queries

#### Show reg keys
```powershell
##show all reg keys
(Gci -Path Registry::).name

##lets take HKEY_CURRENT_USER as a subkey example. Let's see the entries in this subkey
(Gci -Path HKCU:\).name

# If you want to absolutely fuck your life up, you can list the names recursively....will take forever though
(Gci -Path HKCU:\ -recurse).name
```
![image](https://user-images.githubusercontent.com/44196051/119998273-75768c80-bfc8-11eb-869a-807a140d7a52.png)

#### Read a reg entry
```powershell
 Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\SysmonDrv"
```
![image](https://user-images.githubusercontent.com/44196051/119994436-832a1300-bfc4-11eb-98cb-b4148413ac97.png)

#### Remove a reg entry
If there's a malicious reg entry, you can remove it this way
```powershell
# Read the reg to make sure this is the bad boy you want
get-itemproperty -Path 'HKCU:\Keyboard Layout\Preload\'
#remove it by piping it to remove-item
get-itemproperty -Path 'HKCU:\Keyboard Layout\Preload\' | Remove-Item -force
# double check it's gone by trying to re-read it
get-itemproperty -Path 'HKCU:\Keyboard Layout\Preload\'
```
![image](https://user-images.githubusercontent.com/44196051/119999624-d8b4ee80-bfc9-11eb-9770-5ec6e78f9714.png)

## Log Troubleshooting 
I've tended to use these commands to troubleshoot Windows Event Forwarding and other log related stuff

### Show Logs
Show logs that are actually enabled and whose contents isn't empty.
```powershell
Get-WinEvent -ListLog *|
where-object {$_.IsEnabled -eq "True" -and $_.RecordCount -gt "0"} | 
sort-object -property LogName | 
format-table LogName -autosize -wrap
```
![image](https://user-images.githubusercontent.com/44196051/120351284-a96aee00-c2f7-11eb-906d-a8469175b209.png)


#### Overview of what a specific log is up to
```powershell
Get-WinEvent -ListLog Microsoft-Windows-Sysmon/Operational | Format-List -Property * 
```
![image](https://user-images.githubusercontent.com/44196051/120352076-547ba780-c2f8-11eb-8fa7-f8b11f4776b1.png)

#### Specifically get the last time a log was written to
```powershell
(Get-WinEvent -ListLog Microsoft-Windows-Sysmon/Operational).lastwritetime 
```
![image](https://user-images.githubusercontent.com/44196051/119979946-81a41f00-bfb3-11eb-8bc0-f2e893440b18.png)

#### Compare the date and time a log was last written to
Checks if the date was written recently, and if so, just print _sysmon working_ if not recent, then print the date last written. I've found sometimes that sometimes sysmon bugs out on a machine, and stops committing to logs. Change the number after `-ge` to be more flexible than the one day it currently compares to

```powershell
$b = (Get-WinEvent -ListLog Microsoft-Windows-Sysmon/Operational).lastwritetime; 
$a = Get-WinEvent -ListLog Microsoft-Windows-Sysmon/Operational | where-object {(new-timespan $_.LastWriteTime).days -ge 1}; 
if ($a -eq $null){Write-host "sysmon_working"} else {Write-host "$env:computername $b"}
```
![image](https://user-images.githubusercontent.com/44196051/119979908-72bd6c80-bfb3-11eb-9bff-856ebcc01375.png)

#### WinRM & WECSVC permissions
Test the permissions of winrm - used to see windows event forwarding working, which uses winrm usually on endpoints and wecsvc account on servers
```cmd
netsh http show urlacl url=http://+:5985/wsman/ && netsh http show urlacl url=https://+:5986/wsman/
``` 
![image](https://user-images.githubusercontent.com/44196051/119980070-ae583680-bfb3-11eb-8da7-51d7e5393599.png)

## Code Red
### Disconnect network adaptor, firewall the fuck out of an endpoint, and display warning box
This is a code-red command. Used to isolate a machine in an emergency.
Will isolate a machine and display a warning box. 
In the penultimate and final line, you can change the text and title that will pop up for the user
```powershell
New-NetFirewallRule -DisplayName "Block all outbound traffic" -Direction Outbound -Action Block | out-null; 
New-NetFirewallRule -DisplayName "Block all inbound traffic" -Direction Inbound -Action Block | out-null; 
$adapter = Get-NetAdapter|foreach { $_.Name } ; Disable-NetAdapter -Name "$adapter" -Confirm:$false; 
Add-Type -AssemblyName PresentationCore,PresentationFramework; 
[System.Windows.MessageBox]::Show('Your Computer has been Disconnected from the Internet for Security Issues. Please do not try to re-connect to the internet. Contact Security Helpdesk Desk ',' CompanyNameHere Security Alert',[System.Windows.MessageBoxButton]::OK,[System.Windows.MessageBoxImage]::Information)
```
![image](https://user-images.githubusercontent.com/44196051/119979598-0e9aa880-bfb3-11eb-9882-08d02a0d3026.png)

## Powershell Tips
### Get Alias
PwSh is great at abbreviating the commands. Unfortunately, when you're trying to read someone else's abbreviated PwSh it can be ballache to figure out exactly what each weird abbrevation does.

Equally, if you're trying to write something smol and cute you'll want to use abbrevations!

Whatever you're trying, you can use `Get-Alias` to figure all of it out
```powershell
#What does an abbrevation do
get-alias -name gwmi
#What is the abbrevation for this
get-alias -definition write-output
#List all alias' and their full command
get-alias
```
![image](https://user-images.githubusercontent.com/44196051/120551039-81f64d00-c3ed-11eb-8cea-dadb07066942.png)

### Clip
You can pipe straight to your clipboard. Then all you have to do is paste
```powershell
# this will write to terminal
hostname
# this will pipe to clipboard and will NOT write to terminal
hostname | clip
# then paste to test
#ctrl+v
```
![image](https://user-images.githubusercontent.com/44196051/120554093-3e9ddd80-c3f1-11eb-9ddb-d24b8e87481b.png)

### Re-run commands
If you had a command that was great, you can re-run it again from your powershell history!
```powershell
##list out history
get-history
#pick the command you want, and then write down the corresponding number
#now invoke history
Invoke-History -id 38

## You can do the alias / abbrevated method for speed
h
r 43
```
![image](https://user-images.githubusercontent.com/44196051/120559078-48770f00-c3f8-11eb-8726-fd7e627df473.png)
![image](https://user-images.githubusercontent.com/44196051/120559222-8f650480-c3f8-11eb-9b84-ef98dc26cb5c.png)

# Linux
This section is a bit dry, forgive me. My Bash DFIR tends to be a lot more spontaneous and therefore I don't write them down as much as I do the Pwsh one-liners

## Bash History
Checkout the SANS DFIR talk by Half Pomeraz called [You don't know jack about .bash_history](https://www.youtube.com/watch?v=wv1xqOV2RyE). It's a terrifying insight into how weak bash history really is by default

#### Add add timestamps to `.bash_history`
Via .bashrc
```bash
nano ~/.bashrc
#at the bottom
export HISTTIMEFORMAT='%d/%m/%y %T '
#expand bash history size too

#save and exit
source ~/.bashrc
```
Or by /etc/profile
```bash
nano /etc/profile
export HISTTIMEFORMAT='%d/%m/%y %T '

#save and exit
source /etc/profile
```

![image](https://user-images.githubusercontent.com/44196051/119986667-0abf5400-bfbc-11eb-98cf-17d68042250d.png)

Then run the `history` command to see your timestamped bash history

![image](https://user-images.githubusercontent.com/44196051/119987113-9507b800-bfbc-11eb-8033-064c37f5fe26.png)

## Grep and Ack
#### Grep Regex extract IPv4
```bash
grep -E -o "(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)" file.txt | sort | uniq 
```
#### Use Ack to highlight!
One thing I really like about Ack is that it can highlight words easily, which is great for screenshots and reporting. So take the above example, let's say we're looking for two specific IP, we can have ack filter and highlight those

[Ack](https://linux.die.net/man/1/ack) is like Grep's younger, more refined brother. Has some of greps' flags as default, and just makes life a bit easier.

```bash
#install ack if you need to: sudo apt-get install ack
ack -i '127.0.0.1|1.1.1.1' --passthru file.txt
```
![image](https://user-images.githubusercontent.com/44196051/120458382-24331800-c38f-11eb-9527-4c6682be2f5c.png)

## Rapid Malware Analaysis
### Capa
[Capa](https://github.com/fireeye/capa) is a great tool to quickly examine wtf a binary does. This tool is great, it previously helped me identify a keylogger that was pretending to be an update.exe for a program

Usage
```bash
./capa malware.exe > malware.txt
# I tend to do normal run and then verbose
./capa -vv malware.exe >> malware.txt
cat malware.txt
```
![image](https://user-images.githubusercontent.com/44196051/119991809-c1720300-bfc1-11eb-8409-6523a9b0019b.png)

Example of Capa output for the keylogger
![image](https://user-images.githubusercontent.com/44196051/119991358-44df2480-bfc1-11eb-9e6f-23ff445a4900.png)


## Processes and Networks
### Track parent-child processes easier
```bash
ps -aux --forest
```
![image](https://user-images.githubusercontent.com/44196051/120000069-54af3680-bfca-11eb-91a8-221562914878.png)

#### Get a quick overview of network activity
```bash
netstat -plunt
#if you don't have netstat, try ss
ss -plunt
```
![image](https://user-images.githubusercontent.com/44196051/120000196-79a3a980-bfca-11eb-89ed-bbc87b4ca0bc.png)

## Files
### Recursively look for particular file types, and once you find the files get their hashes
Here's the bash alternative
```bash
find . type f -exec sha256sum {} \; 2> /dev/null | grep -Ei '.asp|.js' | sort
```
![image](https://user-images.githubusercontent.com/44196051/120331789-0cec2000-c2e6-11eb-9617-129c9948666b.png)

### Tree
`Tree` is an amazing command. Please bask in its glory. It will recursively list out folders and filders in their parent-child relationship.....or tree-branch relationship I suppose?

```bash
#install sudo apt-get install tree
tree 
```

![image](https://user-images.githubusercontent.com/44196051/120555193-9f79e580-c3f2-11eb-99e4-bf23930e2e54.png)

But WAIT! There's more!

#### Tree and show the users who own the files and directories
```bash
tree -u
#stack this with a grep to find a particular user you're looking for
tree -u | grep 'root'
```
![image](https://user-images.githubusercontent.com/44196051/120555360-de0fa000-c3f2-11eb-8670-fdc522d03418.png)
![image](https://user-images.githubusercontent.com/44196051/120555562-27f88600-c3f3-11eb-891a-98bf39b5cd71.png)

## Bash Tips
### Fixing Mistakes
We all make mistakes, don't worry. Bash forgives you

#### Forget to run as sudo?
We've all done it mate. Luckily, `!!` has your back. The exclamation mark is a history related bash thing. 

Using two exclamations, we can return our previous command. By prefixing `sudo` we are bringing our command back but running it as sudo

```bash
#for testing, fuck up a command that needed sudo but you forgot
cat /etc/shadow
# fix it!
sudo !!
```
![image](https://user-images.githubusercontent.com/44196051/120555899-abb27280-c3f3-11eb-8807-f65b74373ad9.png)

#### Typos in a big old one liner?
The `fc` command is interesting. It gets what was just run in terminal, and puts it in a text editor environment. You can the ammend whatever mistakes you may have made. Then if you save and exit, it will execute your newly ammended command

```bash
##messed up command
cat /etc/prozile
#fix it
fc
#then save and exit
```
![image](https://user-images.githubusercontent.com/44196051/120556440-69d5fc00-c3f4-11eb-98ba-ca1c6ac9d8b5.png)
![image](https://user-images.githubusercontent.com/44196051/120556467-70647380-c3f4-11eb-98a1-4f0dd2fef693.png)

#### Re-run a command in History
If you had a beautiful command you ran ages ago, but can't remember it, you can utilise `history`. But don't copy and paste like a chump. 

Instead, utilise exclamation marks and the corresponding number entry for your command in the history file. This is highlighted in red below

```bash
#bring up your History
history
#pick a command you want to re-run.
# now put one exclamation mark, and the corresponding number for the command you want
!12
```
![image](https://user-images.githubusercontent.com/44196051/120556698-c3d6c180-c3f4-11eb-967d-c5ff873ebb56.png)
