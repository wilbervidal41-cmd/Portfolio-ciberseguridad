# 🛡️ Proyecto 01 — Homelab Blue Team: Proxmox + Wazuh SIEM

> **Wilber Vidal** · Aspirante a Analista SOC / Blue Team  
> Formación: CertiPlus Cybersecurity Expert · Beca Talento Digital (INDOTEL) · Diplomado en Ciberseguridad y Ciberdefensa (Ministerio de Defensa, Rep. Dom.)

---

## 🎯 Resumen

**Problema:** un SOC necesita visibilidad centralizada de lo que ocurre en los equipos de una organización.  
**Qué hice:** construí desde cero, sobre un mini PC de segunda mano, un laboratorio de monitorización de seguridad: hipervisor Proxmox, un SIEM Wazuh y un endpoint Windows 11 reportando eventos en tiempo real.  
**Resultado:** laboratorio operativo con **2 máquinas virtuales**, **1 SIEM single-node (Wazuh 4.14.8)** y **1 agente activo** enviando telemetría. Sirve de base para los proyectos de detección posteriores.

![Agente activo en Wazuh Dashboard](capturas/35-wazuh-agente-activo-dashboard.png)

> 🇬🇧 **English summary:** Built a Blue Team homelab from scratch on refurbished hardware: Proxmox VE hypervisor, a self-hosted Wazuh 4.14.8 SIEM on Ubuntu Server, and a Windows 11 endpoint with an active Wazuh agent. Resolved real deployment issues (geo-blocked CDN, VirtIO drivers, insufficient RAM for the OVA) and documented everything with evidence.

## 💼 Qué demuestra

| Competencia | Evidencia en este proyecto |
|---|---|
| Virtualización (KVM/Proxmox) | Instalación bare-metal, almacenamiento LVM-Thin + directorio de backups, VMs con VirtIO |
| Despliegue de SIEM | Instalación y puesta en marcha de Wazuh (Manager, Indexer, Dashboard) |
| Monitorización de endpoints | Agente Wazuh en Windows 11, enrolado y reportando |
| Redes básicas | Reservas DHCP por MAC, segmentación interna, sin puertos expuestos a Internet |
| Resolución de problemas | 8 incidencias reales diagnosticadas y resueltas (ver tabla abajo) |
| Documentación técnica | Evidencias con capturas y datos sensibles ocultados |

---

## 🏗️ Arquitectura del Laboratorio

**Hardware:** HP EliteDesk 800 G3 DM 35W reciclado (Intel Core i5-7500T, 12 GB RAM, NVMe 256 GB + SSHD 500 GB), gestionado desde una laptop con Xubuntu.

```
Internet
    │
    ▼
Router Claro (10.0.0.1)
    │  DHCP con reservas estáticas por MAC
    │
    ▼
HP EliteDesk 800 G3 — Proxmox VE 9.2.2 (10.0.0.50)
    ├── NVMe 256 GB → local-lvm (LVM-Thin) → VMs + CT
    └── SSHD 500 GB → sshd-backups (Directory) → Backups/ISOs
         │
         ├── VM 100 — Ubuntu Server 26.04.1 LTS
         │     └── Wazuh 4.14.8 Single-Node
         │           ├── Wazuh Manager
         │           ├── Wazuh Indexer (OpenSearch)
         │           └── Wazuh Dashboard (10.0.0.51:443)
         │
         └── VM 101 — Windows 11 Tiny11 (10.0.0.30)
               └── Wazuh Agent 4.14.8 ← ACTIVO
```


---

## 🔧 Stack tecnológico

| Componente | Versión | Rol |
|---|---|---|
| Proxmox VE | 9.2.2 | Hipervisor bare-metal |
| Ubuntu Server | 26.04.1 LTS | SO base para Wazuh |
| Wazuh | 4.14.8 | SIEM / Manager / Indexer / Dashboard |
| Windows 11 | Tiny11 (22621.2283) | Endpoint monitorizado |
| VirtIO | 0.1.302 | Drivers de red KVM para Windows |
| Remmina | — | Cliente RDP desde Xubuntu |


---

## 🧩 Problemas reales resueltos

| # | Problema | Causa | Solución |
|---|---|---|---|
| 1 | `parted: command not found` en Proxmox | Proxmox no incluye parted | Usar `sgdisk` (`apt install gdisk`) |
| 2 | Wazuh OVA: servicios nunca arrancaban | RAM insuficiente (OVA requiere ~6GB libres, host tenía 12GB en total) | Cambiar a instalación manual sobre Ubuntu Server |
| 3 | Error 403 al descargar `wazuh-install.sh` | CDN de Wazuh bloquea IPs del Caribe | Descargar desde Xubuntu (laptop) y transferir con SCP |
| 4 | Script descargado con 0 o 14 bytes | URL incorrecta usada | URL correcta: `https://packages.wazuh.com/4.14/wazuh-install.sh` |
| 5 | Warning: SO no soportado + RAM mínima | Ubuntu 26.04 no está en lista oficial; VM con 2GB al principio | Flag `-i` en el instalador: `sudo bash wazuh-install.sh -a -i` |
| 6 | Sin red en Windows 11 VM post-install | VirtIO NIC requiere driver en Windows | Montar virtio-win ISO, instalar `NetKVM\w11\amd64\netkvm.inf` |
| 7 | Sin portapapeles en noVNC | Limitación del cliente noVNC | Usar Remmina RDP para copiar/pegar comandos largos |
| 8 | `qm terminal 100` → error "no serial interface" | VM OVA sin puerto serie configurado | `qm set 100 -serial0 socket` (aplicado antes de migrar a Ubuntu) |

---

## 🧠 Lo que aprendí

- **Los proveedores de seguridad pueden bloquear regiones:** la descarga de Wazuh devolvía 403 desde República Dominicana; resolví usando otro equipo como relay y transfiriendo por SCP.
- **Dimensionar recursos antes de desplegar:** la OVA de Wazuh no arrancaba con 12 GB de RAM compartidos; migré a una instalación manual sobre Ubuntu Server, más controlable.
- **Virtualización con KVM:** las NIC VirtIO necesitan drivers en Windows (NetKVM); conviene tener la ISO `virtio-win` montada desde el inicio.
- **Eficiencia operativa:** RDP (Remmina) en vez de la consola noVNC para copiar comandos largos; reservas DHCP por MAC en vez de IPs estáticas por VM.

---

## 🔒 Buenas prácticas de seguridad aplicadas

- ✅ Ningún puerto abierto en el router hacia Internet: laboratorio 100 % interno
- ✅ Credenciales fuera del repositorio
- ✅ MACs e IPs públicas/IPv6 ocultas en las capturas
- ✅ Todo el tráfico de monitorización dentro de 10.0.0.0/24

---

## ➡️ Siguiente

El laboratorio sirve de base para el [Proyecto 02 — Atomic Red Team y análisis de detección](../02-atomic-red-team/README.md), donde simulo técnicas de ataque y mido qué detecta el SIEM.

---

<details>
<summary><b>📖 Detalle técnico paso a paso (instalación y configuración)</b></summary>

### Fases de implementación

### Fase 1 — Preparación del Hardware y Proxmox

El equipo es un HP EliteDesk 800 G3 DM de segunda mano, adquirido para el homelab. Se verificó que el procesador tuviera soporte de virtualización por hardware (VT-x) antes de proceder.

![specs](capturas/01-hardware-specs.png)
*Especificaciones del equipo: HP EliteDesk 800 G3 DM 35W, i5-7500T, 12 GB RAM*

Durante el instalador de Proxmox se configuró país/zona horaria y la red de gestión con IP fija `10.0.0.50/24`, gateway y DNS apuntando al router (`10.0.0.1`), hostname `pve.lab.local`.

| Zona horaria y teclado | Red de gestión |
|---|---|
| ![timezone](capturas/02-proxmox-installer-timezone.png) | ![red](capturas/03-proxmox-installer-network.png) |

Tras el primer reinicio se accede a la interfaz web en `https://10.0.0.50:8006`:

![login](capturas/04-proxmox-login.png)

**Verificación de VT-x en Proxmox:**
```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
# Resultado: 8  ← virtualización por hardware habilitada
```

| Guía consultada | Resultado en el host |
|---|---|
| ![guia](capturas/05-verificacion-vtx-guia.png) | ![resultado](capturas/06-verificacion-vtx-resultado.png) |

**Repositorios:** se desactivaron los repositorios *enterprise* (requieren suscripción) y se habilitó `pve-no-subscription` para poder actualizar el sistema (Debian trixie).

![repos](capturas/07-repositorios-no-subscription.png)

Proxmox VE 9.2.2 se instaló desde USB booteable. Se omitió documentar la creación del USB y configuración de BIOS por ser pasos estándar.

---

### Fase 2 — Configuración de Almacenamiento en Proxmox

El equipo tiene dos discos:
- **NVMe 256 GB** → usado para VMs (LVM-Thin, ya configurado por el instalador de Proxmox como `local-lvm`)
- **SSHD 500 GB** → configurado manualmente como almacenamiento de backups e ISOs

Identificación de los discos con `lsblk`: `nvme0n1` (238.5 GB, sistema + LVM) y `sda` (465.8 GB, SSHD sin formatear).

![lsblk](capturas/08-lsblk-discos.png)

**Particionado del SSHD (sin `parted`, usando `sgdisk`):**

> 💡 *Proxmox no incluye `parted` por defecto. Se usó `sgdisk` del paquete `gdisk`.*

```bash
wipefs -a /dev/sda
sgdisk -Z /dev/sda
sgdisk -n 1:0:0 -t 1:8300 /dev/sda
mkfs.ext4 /dev/sda1
```

**Montaje permanente en `/etc/fstab`:**
```bash
mkdir -p /mnt/sshd-backups
echo '/dev/sda1  /mnt/sshd-backups  ext4  defaults  0  2' >> /etc/fstab
mount -a
```

**Registro del storage en Proxmox:**
```
Datacenter → Storage → Add → Directory
  ID: sshd-backups
  Directory: /mnt/sshd-backups
  Content: Disk image, ISO image, Backup
```

![agregar directorio](capturas/09-storage-agregar-directorio.png)

Resultado final con los tres almacenamientos (`local`, `local-lvm`, `sshd-backups`):

![storage final](capturas/10-storage-final.png)

---

### Fase 3 — Red: Reservas DHCP en Router Claro

Para tener IPs fijas sin configurar IPs estáticas en cada VM, se configuraron reservas DHCP en el router Claro del hogar usando las MACs de cada VM.

| Dispositivo | MAC | IP Asignada |
|---|---|---|
| Proxmox Host | (MAC del host) | 10.0.0.50 |
| VM Wazuh (Ubuntu) | (MAC VirtIO) | 10.0.0.51 |
| VM Windows 11 | (MAC VirtIO) | 10.0.0.30 |

![dhcp](capturas/11-router-dhcp-reserva-proxmox.png)
*Reserva DHCP en el router Claro para el host Proxmox (MAC ocultada)*

> ⚠️ *Las MACs reales han sido omitidas de esta documentación por privacidad.*

---

### Fase 4 — VM Ubuntu Server + Wazuh 4.14.8

#### 4.1 Creación de la VM en Proxmox

```
VM 100 — wazuh-server
  CPU: 2 cores
  RAM: 8 GB
  Disco: 80 GB (local-lvm, VirtIO)
  Red: VirtIO (bridge vmbr0)
  ISO: ubuntu-26.04.1-live-server-amd64.iso
```

Ubuntu Server 26.04.1 LTS se instaló con configuración mínima: perfil de usuario `wilber`, hostname `wazuh-server`, sin instalación de paquetes adicionales.

#### 4.2 Problema: CDN de Wazuh bloqueado geográficamente

Al intentar descargar el script de instalación directamente en la VM Ubuntu, el servidor de paquetes de Wazuh devolvía error 403 (bloqueo geográfico en el Caribe).

**Solución:** Descargar desde el laptop Xubuntu y transferir vía SCP a la VM.

```bash
# En Xubuntu (laptop de trabajo):
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
ls -lh wazuh-install.sh
# -rw-rw-r-- 1 wilber ... 211K Oct 8 19:43 wazuh-install.sh ✓

# Transferir a la VM Wazuh:
scp wazuh-install.sh wilber@10.0.0.51:~/
```

> ⚠️ *Intentos fallidos previos descargaron el script con 0 o 14 bytes por usar URLs incorrectas.*

**Capturas del proceso:**

| Captura | Descripción |
|---|---|
| ![descarga error](capturas/21-wazuh-script-descarga-xubuntu-error.png) | Primer intento: script de 14 bytes (URL incorrecta) |
| ![descarga correcto](capturas/22-wazuh-script-descarga-correcto.png) | Descarga correcta: 211KB desde packages.wazuh.com/4.14 |

#### 4.3 Instalación de Wazuh

```bash
# Con flag -i para ignorar advertencias de SO no soportado y RAM mínima
sudo bash wazuh-install.sh -a -i
```

> ℹ️ *Ubuntu 26.04 no está en la lista de sistemas soportados por Wazuh 4.14.8. El flag `-i` (ignore) permite continuar la instalación.*

La instalación tardó aproximadamente 15-20 minutos. Al finalizar, el instalador mostró las credenciales de acceso.

**Capturas de instalación:**

| Captura | Descripción |
|---|---|
| ![inicio instalacion](capturas/23-wazuh-instalacion-inicio.png) | Inicio del script: advertencia de SO y hardware mínimo |
| ![credenciales](capturas/24-wazuh-instalado-credenciales.png) | Instalación completada con credenciales generadas |
| ![dashboard](capturas/25-wazuh-dashboard.png) | Wazuh Dashboard funcionando en https://10.0.0.51 |

```
URL: https://10.0.0.51
Usuario: admin
Contraseña: [NO incluida en este repositorio]
```

---

### Fase 5 — VM Windows 11 Tiny11

#### 5.1 Creación de la VM en Proxmox

```
VM 101 — windows-endpoint
  CPU: 2 cores
  RAM: 4 GB
  Disco: 60 GB (local-lvm, VirtIO SCSI)
  Red: VirtIO (bridge vmbr0)
  ISO: tiny11_2311_x64.iso (montada en CD-ROM)
  CD-ROM 2: virtio-win-0.1.302.iso (drivers VirtIO)
```

**Verificación de RAM disponible en el host antes de crear la VM:**

![ram proxmox](capturas/26-proxmox-ram-disponible.png)
*Proxmox host: 11GB total, 5.7GB usados, 3GB libres + 8GB swap disponibles*

#### 5.2 Instalación de Windows 11 Tiny11

Tiny11 es una versión reducida de Windows 11 ideal para entornos de laboratorio con recursos limitados.

![windows setup](capturas/27-windows-setup.png)
*Pantalla inicial del instalador de Windows*

![windows escritorio](capturas/28-windows-escritorio.png)
*Windows 11 Tiny11 instalado y en el escritorio — 10/8/2026 8:35 PM*

#### 5.3 Problema: Sin red después de instalar Windows

Después de la instalación, Windows no detectaba ningún adaptador de red. Esto es esperado con NICs VirtIO en KVM: Windows necesita el driver `NetKVM`.

**Solución:** Montar la ISO de VirtIO como segundo CD-ROM e instalar el driver.

```
Explorador de archivos → CD Drive (D:) virtio-win-0.1.302
  → NetKVM → w11 → amd64 → netkvm.inf (clic derecho → Instalar)
```

![netkvm driver](capturas/31-windows-netkvm-driver.png)
*Carpeta amd64 de NetKVM en el CD virtio-win — seleccionado netkvm (Security Catalog)*

Después de instalar el driver y reiniciar, la VM obtuvo IP `10.0.0.30` por DHCP (reserva preconfigurada en el router).

---

### Fase 6 — Despliegue del Agente Wazuh en Windows

#### 6.1 Obtener el comando de instalación desde el Dashboard

En Wazuh Dashboard → Endpoints → Deploy new agent → Windows → Windows endpoint

![wazuh deploy cmd](capturas/30-wazuh-deploy-agente-cmd.png)
*Comando generado por el Dashboard: Invoke-WebRequest + msiexec con parámetros WAZUH_MANAGER y WAZUH_AGENT_NAME*

#### 6.2 Copiar el comando desde Xubuntu a la VM Windows

El portapapeles no funciona entre el cliente Xubuntu y la consola noVNC de Proxmox. 

**Solución:** Conectarse a la VM Windows por RDP usando Remmina desde Xubuntu, lo que permite copiar y pegar entre los dos sistemas.

```
Remmina → Nueva conexión RDP
  Host: 10.0.0.30
  Usuario: (usuario Windows)
  Contraseña: (contraseña Windows)
```

![rdp remmina](capturas/33-windows-rdp-remmina.png)
*Conexión RDP desde Xubuntu vía Remmina al Windows endpoint (10.0.0.30)*

#### 6.3 Instalación del agente vía PowerShell

```powershell
# En PowerShell como Administrador en la VM Windows:
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.8-1.msi `
  -OutFile $env:tmp\wazuh-agent; `
  msiexec.exe /i $env:tmp\wazuh-agent /q `
  WAZUH_MANAGER='10.0.0.51' `
  WAZUH_AGENT_NAME='windows-endpoint'

# Iniciar el servicio:
NET START Wazuh
```

![wazuh agent ps](capturas/34-windows-wazuh-agent-activo-ps.png)
*PowerShell: instalación exitosa + "The Wazuh service was started successfully."*

> 💡 *El primer intento falló porque la VM Windows aún no tenía red (driver NetKVM no instalado). Se instaló el driver primero y se reintentó.*

---

### Fase 7 — Verificación Final en Wazuh Dashboard

Resultado final confirmado en el Wazuh Dashboard:

![agente activo](capturas/35-wazuh-agente-activo-dashboard.png)

```
Endpoints → Agents (1)
  ID: 001
  Name: windows-endpoint
  IP: 10.0.0.30
  OS: Microsoft Windows 11 Pro 10.0.22621.2283
  Version: v4.14.8
  Status: ● active
  Cluster node: node01
```

**El agente Wazuh está activo y reportando al Manager. El laboratorio está operativo.** ✅


### 📸 Registro de capturas

| Archivo | Contenido |
|---|---|
| `01-hardware-specs.png` | Especificaciones del HP EliteDesk 800 G3 DM |
| `02-proxmox-installer-timezone.png` | Instalador Proxmox: país, zona horaria y teclado |
| `03-proxmox-installer-network.png` | Instalador Proxmox: red de gestión 10.0.0.50/24 |
| `04-proxmox-login.png` | Pantalla de login de la interfaz web de Proxmox |
| `05-verificacion-vtx-guia.png` | Guía del comando de verificación de virtualización |
| `06-verificacion-vtx-resultado.png` | Resultado de `egrep -c '(vmx\|svm)' /proc/cpuinfo` = 8 |
| `07-repositorios-no-subscription.png` | Repositorios APT: enterprise desactivados, no-subscription activo |
| `08-lsblk-discos.png` | Discos NVMe y SSHD con `lsblk` |
| `09-storage-agregar-directorio.png` | Alta del storage `sshd-backups` |
| `10-storage-final.png` | Almacenamientos finales en Proxmox |
| `11-router-dhcp-reserva-proxmox.png` | Reserva DHCP del host en el router (MAC ocultada) |
| `20-wazuh-script-interno.png` | Código interno del script wazuh-install.sh (función passwords) |
| `21-wazuh-script-descarga-xubuntu-error.png` | Intento fallido: script de 14 bytes desde URL incorrecta |
| `22-wazuh-script-descarga-correcto.png` | Descarga exitosa: 211KB desde packages.wazuh.com/4.14 |
| `23-wazuh-instalacion-inicio.png` | Inicio instalación Wazuh: advertencias de SO y hardware |
| `24-wazuh-instalado-credenciales.png` | Fin de instalación con credenciales de acceso |
| `25-wazuh-dashboard.png` | Wazuh Dashboard operativo en https://10.0.0.51 |
| `26-proxmox-ram-disponible.png` | RAM disponible en el host antes de crear VM Windows |
| `27-windows-setup.png` | Instalador de Windows 11 Tiny11 en la VM |
| `28-windows-escritorio.png` | Windows 11 instalado — escritorio limpio |
| `29-wazuh-dashboard-sesion.png` | Dashboard con menú de sesión admin visible |
| `30-wazuh-deploy-agente-cmd.png` | Comando de despliegue del agente generado por el Dashboard |
| `31-windows-netkvm-driver.png` | Driver NetKVM en la ISO virtio-win — solución sin red |
| `32-windows-wazuh-agent-ps-instalando.png` | PowerShell: instalando agente (con error DNS previo visible) |
| `33-windows-rdp-remmina.png` | Conexión RDP con Remmina al Windows endpoint |
| `34-windows-wazuh-agent-activo-ps.png` | PowerShell: agente instalado + servicio Wazuh iniciado exitosamente |
| `35-wazuh-agente-activo-dashboard.png` | Dashboard: windows-endpoint activo, IP 10.0.0.30, v4.14.8 |

</details>

---

## 📚 Referencias

- [Documentación oficial Wazuh 4.14](https://documentation.wazuh.com/current/)
- [Proxmox VE Wiki](https://pve.proxmox.com/wiki/Main_Page)
- [VirtIO Drivers para Windows en KVM](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/)
- [Tiny11 — Windows 11 compacto para laboratorio](https://github.com/ntdevlabs/tiny11builder)

---

*Proyecto documentado el 8 de octubre de 2026 · Homelab Blue Team · Rep. Dominicana*
