# 🏠 Homelab de Ciberseguridad — Red Segmentada con Proxmox y Active Directory Lab

> Laboratorio casero construido desde cero para practicar redes, virtualización y ciberseguridad ofensiva/defensiva (SOC, Red Team, Blue Team), como preparación para un rol profesional en el área.

> **Nota sobre este repositorio:** las direcciones IP, nombres de host y datos de acceso remoto mostrados aquí son genéricos o se han omitido intencionalmente. Este repositorio documenta **diseño, decisiones de arquitectura y aprendizaje**, no sirve como mapa de acceso a la red real.

---

## 🎯 Objetivo del proyecto

Construir y operar un homelab enfocado en **aprendizaje práctico de ciberseguridad**: pentesting ofensivo, ataques a Active Directory, hacking ético y metodología profesional de reporting — con servicios de conveniencia (DNS local, futura nube privada) como aprendizaje secundario.

**Principio de diseño no negociable:** aislamiento de red real entre el entorno de laboratorio/pruebas y los equipos personales. El laboratorio debe poder "romperse" (intencionalmente, durante las prácticas) sin ningún riesgo para la red personal ni los dispositivos de uso diario.

---

## 🖥️ Inventario de hardware

| Equipo | Rol | CPU | RAM | Almacenamiento | Hipervisor/OS |
|---|---|---|---|---|---|
| Lenovo ThinkCentre M715q | Laboratorio — Active Directory Lab (víctima empresarial), nodo principal | Ryzen 5 PRO 2400GE (4C/8T) | 32GB DDR4 | 128GB SSD M.2 + 500GB SATA SSD (VMs) | Proxmox VE |
| Lenovo ThinkCentre M900 (Tiny) | Laboratorio — Nodo ofensivo (atacante) | Intel i7-6700T (4C/8T, VT-x + VT-d) | 16GB DDR4 | 128GB M.2 + 120GB SSD SATA + 500GB HDD (backup compartido) | Proxmox VE |
| Dell OptiPlex 7040 | Laboratorio — 3er nodo hipervisor (cliente de dominio) | Intel i5-6500T (4C/4T) | 16GB DDR4 | 256GB SSD M.2 | Proxmox VE |
| HP EliteDesk 800 G2 (35W) | Servicios de red — DNS, SIEM (Wazuh), monitoreo (Grafana + NUT para energía/UPS), gestión de contenedores | Intel i5-6500T (4C/4T) | 8GB DDR4 | 256GB NVMe + HDD backup | Debian 13 + Docker |

**Red:** ONT de fibra del ISP → Router GL-iNet (firmware OpenWrt, gestiona VLANs/firewall) → Switch gestionado 8 puertos con 802.1Q VLAN → Patch panel Cat6.

**Energía:** UPS dedicado para los nodos del homelab (no onda senoidal pura — decisión justificada por el tipo de fuente externa de los mini PCs, ver sección de decisiones).

---

## 🌐 Arquitectura de red — segmentación por VLAN

| VLAN | Nombre | Propósito | Equipos |
|---|---|---|---|
| A | Confianza | Equipos personales del usuario | Laptop, PC, extensor WiFi |
| B | Laboratorio | Pentesting / hacking ético — atacante y víctimas | 3 nodos Proxmox |
| C | Servicios | Infraestructura de soporte (DNS, monitoreo) | Servidor de servicios |

Cada VLAN vive en su propia subred `/24` independiente, con zonas de firewall dedicadas en el router. Las reglas por defecto son **deny-by-default** entre zonas, con excepciones explícitas y documentadas (ver abajo).

### Política de firewall (resumen)

| Origen | Destino permitido | Resto |
|---|---|---|
| Confianza | Internet, Laboratorio, Servicios (gestión) | — |
| Laboratorio | Solo Internet | Todo tráfico hacia Confianza y Servicios: **bloqueado explícitamente** |
| Servicios | Solo Internet | Todo tráfico hacia Laboratorio: **bloqueado** |

**Única excepción cross-zona:** una regla puntual que permite tráfico DNS (puerto 53) desde cualquier zona hacia el servidor de DNS local en Servicios — nada más se abre entre zonas.

### Validación empírica del aislamiento

- ✅ Ping desde Laboratorio hacia Confianza: 100% packet loss, rechazo activo confirmado (no timeout silencioso).
- ✅ Ping desde Laboratorio hacia Servicios: bloqueado.
- ✅ Resolución DNS desde Laboratorio hacia el servidor de Servicios: funcional (única excepción permitida).
- ✅ Acceso de administración (SSH / panel de hipervisor) desde Confianza hacia Laboratorio: funcional, para gestión.

**Limitación de diseño aceptada conscientemente:** los servicios de monitoreo en la zona Servicios no pueden monitorear activamente los nodos de Laboratorio, por el mismo firewall que los protege. Se decidió no abrir excepciones adicionales, priorizando el aislamiento sobre la conveniencia de un dashboard centralizado.

---

## 🧰 Servicios desplegados

| Servicio | Función |
|---|---|
| Hipervisor de contenedores (Portainer) | Gestión de contenedores Docker |
| Resolver DNS local (Pi-hole) | DNS para toda la red, filtrado de publicidad/tracking |
| SIEM (Wazuh) | Recolección y correlación de logs/alertas de seguridad — pieza central del lado Blue Team/SOC del homelab |
| Monitoreo de métricas (Grafana + Prometheus) | Dashboards de infraestructura: batería del UPS, consumo en watts, CPU, RAM |
| NUT Server (Network UPS Tools) | Monitoreo del UPS — estado de batería, consumo en vatios y apagado automático coordinado de los nodos ante corte eléctrico |

---

## 🗺️ Roadmap de aprendizaje

**Fase 1 — Fundamentos ofensivos:** Nmap, explotación web (DVWA, OWASP Juice Shop), Metasploitable2.

**Fase 2 — Construcción del AD Lab:** Domain Controller, usuarios con privilegios variados, clientes Windows unidos al dominio, configuraciones vulnerables intencionales (contraseñas débiles, SPNs mal configurados, ACLs laxas).

**Fase 3 — Ataques a Active Directory:** BloodHound, Kerberoasting, AS-REP Roasting, Pass-the-Hash/Pass-the-Ticket, movimiento lateral, escalación a Domain Admin.

**Fase 4 — Metodología profesional:** redacción de reportes de pentesting (hallazgo/severidad/evidencia/remediación), mapeo a MITRE ATT&CK, referencia a PTES/OWASP Testing Guide.

**Fase 5 — Snapshots/backups:** uso de snapshots para resetear el entorno vulnerable entre sesiones de práctica.

---

## 🧠 Decisiones de arquitectura clave (con su justificación)

| Decisión | Alternativa descartada | Razón |
|---|---|---|
| Especialización por nodo: un equipo = atacante, otro = víctima empresarial | Consolidar todo en un solo nodo | Refleja un engagement de pentesting real; evita competencia de recursos entre atacante y víctima |
| Segmentación VLAN estricta con deny-by-default | Red plana única | El principio no negociable del proyecto: el laboratorio debe poder romperse sin riesgo para equipos personales |
| Backups en HDD mecánico en vez de SSD | Usar el SSD más rápido disponible | El cuello de botella real de un backup nocturno es irrelevante en minutos; el SSD se reserva para I/O activo de VMs |
| No clúster Proxmox entre nodos de laboratorio | Cluster con quorum device | El objetivo es aislamiento atacante-víctima, no alta disponibilidad; clustering no resuelve el problema real que se buscaba resolver |

---

## 🛠️ Stack técnico

`Proxmox VE` · `Debian` · `Docker` · `pfSense/OpenWrt (firewall)` · `802.1Q VLAN` · `Pi-hole` · `Active Directory` · `Kali Linux` · `BloodHound` · `Metasploit`

---

## 📌 Sobre este repositorio

Este repo documenta el *diseño* y las *decisiones técnicas* del laboratorio como parte de mi portafolio de aprendizaje en ciberseguridad. No incluye direcciones IP reales, credenciales, claves de acceso remoto, ni información suficiente para ubicar o acceder a la red real.
