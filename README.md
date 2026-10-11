# 🛡️ Wilber Vidal — Portfolio de Ciberseguridad (Blue Team)

Aspirante a **Analista SOC / Blue Team**. Construyo y documento un laboratorio de defensa sobre hardware reciclado: monitorización con SIEM, simulación de ataques, ingeniería de detección y respuesta a incidentes. Cada proyecto incluye evidencias, problemas reales y resultados medibles.

**Formación:** CertiPlus Cybersecurity Expert · Beca Talento Digital en Ciberseguridad, nivel intermedio y avanzado (INDOTEL) · Diplomado en Ciberseguridad y Ciberdefensa (Ministerio de Defensa, Rep. Dom.)

> 🇬🇧 **English:** Blue Team / SOC analyst candidate building a hands-on detection lab (Proxmox, Wazuh SIEM, Atomic Red Team, MITRE ATT&CK). Each project is documented with evidence, root-cause analysis and measurable results. Spanish and English speaker.

---

## 📁 Proyectos

| # | Proyecto | Resultado | Tecnologías |
|---|---|---|---|
| 01 | [Homelab Proxmox + Wazuh SIEM](01-proxmox-wazuh/README.md) | Laboratorio operativo: 2 VMs, SIEM propio y endpoint Windows monitorizado | Proxmox, Wazuh, Ubuntu Server, Windows 11 |
| 02 | [Atomic Red Team: análisis de detección](02-atomic-red-team/README.md) | 3 técnicas ATT&CK emuladas: 2 detectadas y 1 brecha documentada con causa raíz | Atomic Red Team, MITRE ATT&CK, Wazuh, PowerShell, auditoría de Windows |
| 03 | Sysmon + reglas de detección propias | *En preparación* | Sysmon, Wazuh rules/decoders |
| 04 | Hardening CIS con puntuación antes/después | *Planificado* | CIS Benchmark, Wazuh SCA |
| 05 | Ataque → detección → respuesta activa | *Planificado* | Kali Linux, Wazuh Active Response |
| 06 | Informe de incidente estilo SOC | *Planificado* | Línea de tiempo, MITRE ATT&CK, IOCs |
| 07 | Detección de red | *Planificado* | Suricata |

---

## 🧰 Competencias demostradas

| Área | Detalle |
|---|---|
| SIEM y monitorización | Despliegue y administración de Wazuh, búsquedas DQL, análisis de alertas |
| Detección y MITRE ATT&CK | Emulación de técnicas, mapeo a tácticas, análisis de brechas |
| Windows | Eventos de seguridad, auditoría (`auditpol`), PowerShell Script Block Logging |
| Virtualización e infraestructura | Proxmox/KVM, almacenamiento LVM-Thin, Ubuntu Server |
| Redes básicas | DHCP, segmentación, laboratorio sin exposición a Internet |
| Metodología | Diagnóstico por capas, resolución de problemas, documentación técnica |

---

*Laboratorio en continua evolución. Todos los datos sensibles (credenciales, MACs, IPs públicas) se omiten de la documentación.*
