# Arquitectura de Red Segura — Empresa

Realizado por:
* Mariana Ruge Vargas
* Laura Sofía Londoño Pérez

## Descripción general

Este proyecto presenta el diseño de una infraestructura de red segura para una empresa, segmentada en zonas de seguridad diferenciadas según el nivel de confianza y criticidad de los activos que contienen. El objetivo es minimizar la superficie de ataque, controlar rigurosamente los flujos de tráfico entre zonas y proteger los datos y sistemas más sensibles mediante una estrategia de defensa en profundidad.

El diagrama completo de la arquitectura se encuentra en el archivo `diagrama-red-segura.png` (o el formato correspondiente) adjunto a este repositorio.

---

## Zonas de la arquitectura

### 1. Internet

Zona no confiable que representa el origen del tráfico externo: usuarios, clientes, proveedores y navegación pública. Todo el tráfico que ingresa desde aquí se considera potencialmente hostil y debe pasar por múltiples controles antes de llegar a cualquier recurso interno.

**Componentes:**
- **Firewall Perimetral (NGFW + IPS + filtrado web):** primer punto de control; inspecciona y filtra el tráfico entrante y saliente, bloqueando amenazas conocidas.
- **VPN Gateway:** permite el acceso remoto seguro mediante autenticación multifactor (MFA), cifrando las comunicaciones de empleados o administradores externos.
- **WAF / Anti-DDoS:** protege las aplicaciones web publicadas contra ataques de capa 7 y denegación de servicio distribuido.

### 2. DMZ Externa

Zona intermedia y desmilitarizada que aloja los servicios que deben ser accesibles desde Internet, aislándolos de la red interna. Ningún host de esta zona tiene acceso directo a la Intranet.

**Componentes:**
- **Balanceador / Reverse Proxy:** distribuye la carga, termina TLS y realiza *health checks*, exponiendo únicamente los puertos publicados.
- **Web / Portal Público:** servicios web accesibles desde Internet (portal corporativo, aplicaciones públicas).
- **DNS / Mail Relay:** servicios de resolución de nombres y correo publicados de forma controlada.
- **IDS/IPS de DMZ:** monitorea el tráfico de la zona en busca de patrones de intrusión y bloquea amenazas en tiempo real.

### 3. DMZ Interna / Gestión

Zona de tránsito entre la DMZ externa y los recursos internos críticos, protegida por un **Firewall Interno** que aplica segmentación y políticas de tráfico este-oeste. Solo el tráfico previamente autorizado desde la DMZ externa puede llegar hasta aquí.

**Componentes:**
- **Servidores de Aplicación:** exponen APIs y servicios internos consumidos por la DMZ externa y la zona empresarial.
- **Zona de Gestión (Jump Server + SIEM + NMS):** subzona de alto privilegio para la administración y supervisión de la infraestructura. El *jump server* centraliza y audita todo acceso administrativo; el SIEM correlaciona eventos y logs de seguridad de toda la red; el NMS monitorea la disponibilidad y desempeño de los dispositivos de red. Al ser un punto de alto riesgo, cuenta con controles de acceso reforzados.
- **Backups:** repositorios de respaldo aislados e inmutables, alimentados desde la Zona de Gestión.
- **PKI / IAM:** gestión de certificados digitales, autenticación, autorización y control centralizado de identidades (AAA).

### 4. Intranet / Zona Empresarial

Red corporativa interna donde operan los usuarios finales, con autenticación centralizada vía Active Directory.

**Componentes:**
- **Intranet — Usuarios:** equipos de cómputo, Wi-Fi corporativo e impresoras, autenticados mediante 802.1X/ACL.
- **Zona Empresarial:** sistemas de negocio como ERP, repositorios de archivos y herramientas de colaboración, comunicados con los servidores de aplicación mediante reglas App/DB.
- **Active Directory:** provee autenticación centralizada para todos los dispositivos de la zona.
- **Autenticación / Endpoints:** soluciones de verificación de identidad y protección de dispositivos finales (EDR).

### 5. Extranet — Terceros

Zona destinada a proveedores y socios externos, con acceso limitado y controlado mediante VPN o ZTNA (Zero Trust Network Access) hacia la Zona de Gestión, evitando exponerlos directamente a la red interna.

### 6. Zona Restringida

Zona de máxima seguridad que contiene datos críticos y bases de datos sensibles. El acceso desde la Zona de Gestión requiere flujos explícitamente aprobados.

**Componentes:**
- **Zona Restringida:** datos críticos y bases sensibles de la organización.
- **HSM / Servidor Crítico:** almacena claves criptográficas y activos de máxima criticidad, protegido con MFA y el principio de privilegio mínimo.

---

## Medidas de seguridad implementadas

| Medida | Propósito |
|---|---|
| **Zero Trust** | Ningún flujo de tráfico se confía por defecto; toda comunicación entre zonas se valida explícitamente. |
| **MFA (autenticación multifactor)** | Refuerza el acceso remoto (VPN) y el acceso a activos críticos (HSM). |
| **Mínimo privilegio** | Los usuarios y sistemas solo acceden a los recursos estrictamente necesarios. |
| **VLAN/VRF** | Segmentación lógica del tráfico entre zonas. |
| **ACL (listas de control de acceso)** | Restringen qué tráfico puede circular entre segmentos. |
| **NGFW/IPS** | Firewalls de nueva generación con prevención de intrusiones en el perímetro y entre zonas internas. |
| **WAF** | Protección específica para aplicaciones web publicadas. |
| **EDR** | Detección y respuesta ante amenazas en los dispositivos finales. |
| **SIEM/SOC** | Correlación de eventos y monitoreo centralizado de seguridad. |
| **Cifrado TLS/IPsec** | Protege la confidencialidad e integridad de los datos en tránsito. |
| **Backups 3-2-1** | Estrategia de respaldo (3 copias, 2 medios distintos, 1 fuera de sitio) para garantizar recuperación ante incidentes. |
| **Registro y auditoría** | Trazabilidad de accesos y eventos para cumplimiento y análisis forense. |
| **Alta disponibilidad** | Redundancia en componentes críticos para asegurar continuidad operativa. |

---

## Justificación del diseño

La segmentación en zonas concéntricas —de menor a mayor confianza (Internet → DMZ externa → DMZ interna/Gestión → Intranet/Restringida)— sigue el principio de **defensa en profundidad**: cada zona actúa como una barrera adicional, de modo que comprometer un segmento no implica acceso directo a los demás.

- La **DMZ externa** aísla los servicios públicos, evitando que un ataque exitoso contra el portal web o el correo comprometa directamente la red interna.
- El **Firewall Interno** entre la DMZ externa y la DMZ interna/Gestión garantiza que solo tráfico explícitamente autorizado avance hacia los sistemas de administración y aplicación.
- La **Zona de Gestión** se mantiene separada de los servidores de aplicación porque concentra accesos administrativos de alto privilegio (jump server, SIEM, NMS); aislarla reduce el riesgo de movimiento lateral en caso de compromiso.
- La **Extranet** se conecta únicamente mediante VPN/ZTNA a la Zona de Gestión, permitiendo colaborar con terceros sin exponerlos a la red interna completa.
- La **Zona Restringida** y el **HSM** están al final de la cadena de confianza, accesibles solo mediante flujos aprobados y MFA, protegiendo los activos de mayor criticidad (datos sensibles y claves criptográficas).
- Los **controles transversales** (Zero Trust, MFA, mínimo privilegio, cifrado, SIEM, backups, auditoría) se aplican de forma consistente en todas las zonas, reforzando la seguridad de extremo a extremo en lugar de depender de un único punto de control.

Este enfoque busca equilibrar la disponibilidad de servicios hacia Internet y terceros con la protección estricta de los activos internos y críticos, cumpliendo con buenas prácticas de arquitectura de seguridad empresarial.

---

## Autor
Actividad realizada como estudiante de Seguridad Informática — Diseño de infraestructura de red segura.
