# 🎯 Proyecto 02 — Atomic Red Team: ¿qué detecta realmente mi SIEM?

> **Wilber Vidal** · Aspirante a Analista SOC / Blue Team  
> Formación: CertiPlus Cybersecurity Expert · Beca Talento Digital (INDOTEL) · Diplomado en Ciberseguridad y Ciberdefensa (Ministerio de Defensa, Rep. Dom.)

---

## 🎯 Resumen

**Problema:** tener un SIEM no garantiza detectar ataques; hay que medir qué ve y qué se le escapa.  
**Qué hice:** sobre el laboratorio del [Proyecto 01](../01-proxmox-wazuh/README.md), emulé **3 técnicas de MITRE ATT&CK** con Atomic Red Team en un endpoint Windows 11 y analicé, técnica por técnica, si Wazuh generaba alerta, con qué regla y con qué mapeo MITRE.  
**Resultado:** **2 de 3 técnicas detectadas** (una de ellas tras corregir la auditoría de Windows, que era la causa real del fallo) y **1 brecha de detección documentada** con su causa raíz, que se aborda en el Proyecto 03 con reglas personalizadas.

![Antes: sin detección](capturas/06-t1053-005-sin-deteccion.png)
![Después: detectado tras habilitar la auditoría](capturas/09-wazuh-t1053-005-detectado.png)

> 🇬🇧 **English summary:** Emulated 3 MITRE ATT&CK techniques (T1136.001, T1053.005, T1059.001) with Atomic Red Team against a Windows 11 endpoint monitored by Wazuh. Two were detected (one only after fixing a disabled Windows audit policy); one is a documented rule gap: PowerShell script block events reach the manager but no default rule alerts on them. Includes root-cause analysis across the four layers of the pipeline (source, collection, transport, rules) and a false-positive investigation.

## 💼 Qué demuestra

| Competencia | Evidencia |
|---|---|
| Emulación de adversarios | Atomic Red Team: ejecución controlada y limpieza de técnicas ATT&CK |
| MITRE ATT&CK | Mapeo de cada prueba a táctica y técnica; contraste con el mapeo automático de Wazuh |
| Análisis de brechas de detección | Diagnóstico por capas: origen → recolección → transporte → reglas |
| Telemetría de Windows | Eventos 4720, 4698, 4104; `auditpol`; Script Block Logging |
| Administración de Wazuh | Configuración del agente (`ossec.conf`, eventchannel), búsquedas DQL, `archives.json` |
| Triage de alertas | Identificación de un falso positivo (regla 91816) originado por los chequeos SCA |

---

## 🧰 Entorno

| Elemento | Detalle |
|---|---|
| Hipervisor | Proxmox VE 9.2.2 (10.0.0.50) |
| SIEM | Wazuh 4.14.8 single-node (VM 100, 10.0.0.51) |
| Endpoint atacado | Windows 11 Tiny11 (VM 101, `windows-endpoint`, 10.0.0.30, agente 001) |
| Herramienta | Invoke-AtomicRedTeam + biblioteca de atomics |
| Protección | Snapshot `Pre-atomic-red-team` (con RAM) antes de empezar |

> ⚠️ Las simulaciones se ejecutaron solo en una VM de laboratorio aislada, con snapshot previo. La exclusión de Windows Defender sobre `C:\AtomicRedTeam` es exclusiva del laboratorio.

![Snapshot previo en Proxmox](capturas/01-snapshot-proxmox.png)

---

<details>
<summary><b>🧪 Metodología e instalación de Atomic Red Team</b></summary>

Para cada técnica se siguió el mismo ciclo:

1. **Predicción:** anotar si se espera que Wazuh alerte o no (y por qué).
2. **Ejecución:** `Invoke-AtomicTest <técnica> -TestNumbers <n>`.
3. **Observación:** Wazuh → *Threat Hunting → Events*, filtrando por `agent.name:windows-endpoint`.
4. **Análisis:** regla disparada, nivel, evento de Windows de origen y mapeo MITRE.
5. **Limpieza:** `Invoke-AtomicTest ... -Cleanup` para revertir los cambios.

> Distinción clave: un **evento** (Windows lo registra) no es una **alerta** (una regla de Wazuh lo considera relevante). Solo los eventos que coinciden con una regla aparecen en el Dashboard.

Instalación (resumen):

```powershell
Add-MpPreference -ExclusionPath "C:\AtomicRedTeam"      # solo laboratorio
Set-ExecutionPolicy Bypass -Scope CurrentUser -Force
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)
Install-AtomicRedTeam -getAtomics -Force
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
```

</details>

---

## 📊 Resumen de resultados

| # | Técnica MITRE ATT&CK | Test | Resultado inicial | Causa / acción | Resultado final |
|---|---|---|---|---|---|
| 1 | **T1136.001** Create Account: Local Account (Persistence) | T1136.001-4 (`net user /add`) | ✅ Detectado | Evento 4720 → familia de reglas 601xx; la regla 60109 se asocia a **T1098** (Account Manipulation) | ✅ Detectado (mapeo MITRE parcial) |
| 2 | **T1053.005** Scheduled Task (Execution/Persistence/Priv. Esc.) | T1053.005-2 (`schtasks /create`) | ❌ Sin alerta | Auditoría "Other Object Access Events" = *No Auditing* → Windows no generaba el evento 4698 | ✅ Detectado tras `auditpol` (regla 60228, nivel 4, **T1053**) |
| 3 | **T1059.001** PowerShell (Execution) | T1059.001-17 (`Write-Host`) | ❌ Sin alerta | Script Block Logging desactivado y canal PowerShell/Operational no recolectado; corregido, pero el evento llega y **ninguna regla lo convierte en alerta** | ⚠️ Brecha de reglas (se aborda en el Proyecto 03) |

---

## 🔍 Detalle de cada prueba

### 1. T1136.001 — Creación de cuenta local (persistencia)

Un atacante que consigue privilegios crea una cuenta propia para volver cuando quiera. El test ejecuta `net user /add T1136.001_CMD`.

![Lista de tests T1136.001](capturas/02-lista-t1136-001.png)
![Ejecución del test](capturas/03-t1136-001-ejecucion.png)

Wazuh generó 4 eventos (creación, cambio y borrado de la cuenta durante el ciclo de ejecución + limpieza):

![Eventos en Wazuh](capturas/04-wazuh-t1136-001-eventos.png)

La alerta de creación proviene del evento de Windows **4720** (*A user account was created*) y la regla 60109 la etiqueta con **T1098 – Account Manipulation / Persistence**. Es un mapeo cercano pero no exacto: la técnica emulada es T1136.001.

![Regla 60109 y MITRE](capturas/05-regla-60109-mitre-t1098.png)

**Lectura del analista:** la detección es buena porque Windows audita la gestión de cuentas por defecto. Un SOC vería quién creó la cuenta (`subjectUserName`) y cuál (`targetUserName`).

### 2. T1053.005 — Tarea programada (antes y después)

**Antes:** el test creó la tarea `spawn`, pero Wazuh no mostró nada.

![Sin detección](capturas/06-t1053-005-sin-deteccion.png)

**Diagnóstico:** el evento 4698 (*A scheduled task was created*) pertenece a la subcategoría de auditoría *Other Object Access Events*, que estaba en *No Auditing*. Si Windows no registra el evento, el SIEM no puede alertar.

![auditpol: No Auditing](capturas/07-auditpol-no-auditing.png)

**Corrección:**

```powershell
auditpol /set /subcategory:"Other Object Access Events" /success:enable /failure:enable
```

![auditpol habilitado](capturas/08-auditpol-habilitado.png)

**Después:** al repetir el test, Wazuh dispara la regla **60228** (*A scheduled task was created*), y además la regla 60112 por el cambio de política de auditoría.

![Detectado tras habilitar la auditoría](capturas/09-wazuh-t1053-005-detectado.png)
![Regla 60228 y MITRE T1053](capturas/10-regla-60228-mitre-t1053.png)

**Lectura del analista:** la detección depende de la **configuración de auditoría del endpoint**. Un hardening básico (CIS) incluye habilitarla.

### 3. T1059.001 — PowerShell (brecha de detección documentada)

El test ejecuta un comando PowerShell sencillo (`Write-Host 'Hello, from PowerShell!'`).

![Ejecución del test](capturas/11-t1059-001-ejecucion.png)

Se investigó capa por capa, cada una con su evidencia:

| Capa | Pregunta | Resultado |
|---|---|---|
| 1. Origen | ¿Windows registra el comando? | Al principio no. Se activó **Script Block Logging** (política de registro `EnableScriptBlockLogging=1`). Tras ello aparece el evento **4104** en `Microsoft-Windows-PowerShell/Operational`. |
| 2. Recolección | ¿El agente Wazuh lee ese canal? | No por defecto. Se añadió un bloque `<localfile>` con `eventchannel` en `ossec.conf` y se reinició el agente (verificado en `ossec.log`). |
| 3. Transporte | ¿Llega el evento al manager? | **Sí.** Se activó temporalmente `logall_json` en el manager y el evento apareció en `archives.json`, decodificado (`windows_eventchannel`) con el `scriptBlockText` completo. |
| 4. Reglas | ¿Alguna regla lo convierte en alerta? | **No.** Ningún conjunto de reglas por defecto alerta sobre este script benigno. |

![Evento 4104 en Windows](capturas/12-evento-4104-en-windows.png)
![Evento en archives.json del manager, sin alerta](capturas/13-archives-json-evento-sin-alerta.png)

```powershell
# Bloque añadido a ossec.conf (agente Windows)
<localfile>
  <location>Microsoft-Windows-PowerShell/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

**Conclusión:** es una brecha de **reglas**, no de recolección. Detectar PowerShell malicioso exigirá reglas propias (por ejemplo, sobre `-EncodedCommand`, `IEX`, `DownloadString`), lo cual se desarrolla en el Proyecto 03. Tras la prueba, `logall_json` se desactivó y el archivo se eliminó.

---

## ⚠️ Falsos positivos observados

Al activar el canal de PowerShell aparecieron alertas **91816** (*PowerShell script querying system environment variables*, nivel 4, MITRE **T1082 – System Information Discovery**). Parecían ataques, pero se originaban en los **chequeos SCA de Wazuh** (`secedit` ejecutado por PowerShell), no en las pruebas. Se verificó leyendo el `scriptBlockText` del evento.

![Falso positivo 91816](capturas/14-falso-positivo-91816-sca.png)

**Aprendizaje:** una alerta con técnica MITRE no es automáticamente un ataque; hay que leer el evento original y correlacionar con la actividad conocida.

---

## ❌ Problemas encontrados y soluciones

| Problema | Causa | Solución |
|---|---|---|
| T1053.005 sin alerta | Auditoría *Other Object Access Events* desactivada | `auditpol /set ...` |
| Sin eventos 4104 | Script Block Logging desactivado | Política en el registro de Windows |
| Canal PowerShell no leído | El agente no lo recolecta por defecto | `<localfile>` + `eventchannel` en `ossec.conf` |
| Agente detenido tras editar `ossec.conf` | Bloque pegado dentro de otro bloque XML | Restaurar el `.bak` e insertar el bloque antes del `</ossec_config>` final |
| Alertas 91816 atribuidas al test | Venían de los chequeos SCA | Comprobar el contenido del script block |
| Horas que no coinciden | Wazuh en UTC, Windows en hora local (UTC-4) | Convertir antes de comparar |

---

## 🧠 Conclusiones

1. **Visibilidad no es detección.** El fallo puede estar en el origen (auditoría), la recolección (agente), el transporte (manager) o las reglas; hay que descartarlas en orden.
2. **La configuración del endpoint importa tanto como el SIEM:** sin auditoría ni logging, el SIEM está ciego.
3. **El mapeo MITRE automático es parcial** y debe contrastarse con la técnica real.
4. **Las alertas se validan:** un falso positivo se descubre leyendo el evento original.
5. **Documentar lo que NO se detecta** es lo que convierte un laboratorio en un análisis de brechas.

---

## ➡️ Siguiente

**Proyecto 03:** Sysmon + reglas y decodificadores personalizados en Wazuh para cerrar las brechas encontradas (PowerShell codificado, ajuste del falso positivo 91816, mayor severidad para tareas programadas).
