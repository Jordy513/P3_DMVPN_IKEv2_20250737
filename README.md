# VPN Hub-and-Spoke — DMVPN Fase 3 con IKEv2

### Jordy Jose Rosario Ortiz · Matrícula: 2025-0737

**Seguridad de Redes 2026-C-2 · ITLA**

---

## 📋 Tabla de Contenido

1. [Objetivo del Laboratorio](#1-objetivo-del-laboratorio)
2. [Marco Teórico](#2-marco-teórico)
   - [DMVPN Fase 3 — Shortcut Switching](#21-dmvpn-fase-3--shortcut-switching)
   - [Fase 2 vs. Fase 3](#22-fase-2-vs-fase-3)
   - [OSPF en DMVPN — Por qué no EIGRP](#23-ospf-en-dmvpn--por-qué-no-eigrp)
   - [IKEv2 en DMVPN](#24-ikev2-en-dmvpn)
   - [Parámetros Criptográficos Utilizados](#25-parámetros-criptográficos-utilizados)
3. [Documentación de la Red](#3-documentación-de-la-red)
   - [Topología](#31-topología)
   - [Tabla de Dispositivos y Direccionamiento IP](#32-tabla-de-dispositivos-y-direccionamiento-ip)
4. [Scripts de Configuración](#4-scripts-de-configuración)
   - [Router R1 — HUB](#41-router-r1--hub)
   - [Router R2 — SPOKE 1](#42-router-r2--spoke-1)
   - [Router R3 — SPOKE 2](#43-router-r3--spoke-2)
   - [Configuración de PCs (Hosts)](#44-configuración-de-pcs-hosts)
5. [Verificación del Túnel](#5-verificación-del-túnel)
6. [Capturas de Pantalla](#6-capturas-de-pantalla)
7. [Consideraciones de Seguridad](#7-consideraciones-de-seguridad)
8. [Video Demostrativo](#8-video-demostrativo)
9. [Referencias](#9-referencias)

---

## 1. Objetivo del Laboratorio

El objetivo de este laboratorio es **implementar y verificar una VPN Hub-and-Spoke punto a multipunto utilizando DMVPN Fase 3 con IKEv2 y OSPF** sobre infraestructura Cisco IOS. A través de esta práctica se busca demostrar:

* La configuración de DMVPN Fase 3 mediante `ip nhrp redirect` en el Hub e `ip nhrp shortcut` en los Spokes, que es el mecanismo que diferencia la Fase 3 de la Fase 2.
* El uso de IKEv2 como protocolo de negociación de claves, reemplazando el bloque `crypto isakmp` de IKEv1 por la jerarquía `crypto ikev2 proposal` / `crypto ikev2 policy` / `crypto ikev2 keyring` / `crypto ikev2 profile`.
* La propagación de rutas mediante **OSPF** con `ip ospf network broadcast` sobre la nube DMVPN, donde el Hub es forzado a ser el DR (Designated Router) con `ip ospf priority 2` y los Spokes con `ip ospf priority 0`, eliminando los problemas de next-hop-self inherentes a EIGRP en entornos DMVPN.
* La verificación de comunicación Spoke-to-Spoke directa mediante la tabla de routing optimizada por NHRP Shortcut, demostrando la ventaja de Fase 3 sobre Fase 2.

---

## 2. Marco Teórico

### 2.1 DMVPN Fase 3 — Shortcut Switching

DMVPN Fase 3 es la evolución de Fase 2. Mientras que en Fase 2 la resolución Spoke-to-Spoke la dispara el propio Spoke al detectar que el next-hop del paquete es diferente al next-hop de NHRP, en **Fase 3 es el Hub quien toma la iniciativa**.

El mecanismo funciona así:

```
1. PC2 hace ping a PC3. El paquete llega al Hub (R1) porque
   OSPF anunció esa ruta vía el Hub.

2. El Hub detecta que ese paquete podría ir directamente
   a R3 sin pasar por él → envía un NHRP Redirect a R2
   diciéndole: "para llegar a 20.25.37.128/26, ve directo
   a 10.25.37.3, no pases por mí".

3. R2 recibe el Redirect, instala una ruta NHRP Shortcut
   en su tabla de routing (marcada con H), y los paquetes
   siguientes van directo a R3.

4. IKEv2 cifra ese túnel directo Spoke-to-Spoke.
```

La diferencia clave con Fase 2: en Fase 2 los Spokes también usan mGRE y resuelven el next-hop por sí solos. En Fase 3 el Hub es quien **redirige activamente**, lo que permite usar protocolos de routing más simples en los Spokes (el Hub resume todo) y escala mejor con muchos Spokes.

### 2.2 Fase 2 vs. Fase 3

| Característica | DMVPN Fase 2 | DMVPN Fase 3 |
|---|---|---|
| **Comando Hub** | Solo `ip nhrp map multicast dynamic` | Agrega `ip nhrp redirect` |
| **Comando Spoke** | Solo registro NHRP básico | Agrega `ip nhrp shortcut` |
| **Quién inicia la resolución** | El Spoke origen (detecta next-hop diferente) | El Hub (envía NHRP Redirect al Spoke) |
| **Ruta instalada en el Spoke** | NHRP dinámico en caché | Ruta `H` (NHRP Shortcut) en tabla de routing |
| **Protocolo de routing** | Requiere `no ip next-hop-self` (EIGRP) o broadcast (OSPF) | Funciona con cualquier protocolo, el Hub resume todo |
| **Escalabilidad** | Media | Alta — el Hub controla la redirección |
| **Visibilidad** | `show ip nhrp` | `show ip route` (aparece como `H`) |

### 2.3 OSPF en DMVPN — Por qué no EIGRP

EIGRP en DMVPN tiene un problema estructural: por defecto hace **next-hop-self**, reemplazando el next-hop real de cada Spoke por la IP del Hub al redistribuir rutas. Esto obliga a configurar `no ip next-hop-self eigrp` en el Hub y complica la convergencia en Fase 2.

OSPF resuelve esto con una configuración más limpia:

* **`ip ospf network broadcast`** en todas las interfaces Tunnel0 — hace que OSPF trate la nube mGRE como una red broadcast, permitiendo elección de DR/BDR.
* **`ip ospf priority 2`** en el Hub — garantiza que R1 siempre sea el DR.
* **`ip ospf priority 0`** en los Spokes — los excluye de la elección de DR/BDR.
* El Hub como DR anuncia las rutas de cada Spoke al resto de la nube sin modificar el next-hop, lo que en Fase 3 permite que el Hub envíe NHRP Redirects correctos.

### 2.4 IKEv2 en DMVPN

En DMVPN con IKEv2, la PSK wildcard del Hub (`address 0.0.0.0 0.0.0.0`) se configura de forma diferente al IKEv1. En IKEv2 se usa el keyring con un peer `any`:

```cisco
crypto ikev2 keyring KR_DMVPN
 peer SPOKES_ANY
  address 0.0.0.0 0.0.0.0
  pre-shared-key local  DmvpnITLA2026!
  pre-shared-key remote DmvpnITLA2026!
```

Y el profile usa `match identity remote address 0.0.0.0` para aceptar cualquier Spoke:

```cisco
crypto ikev2 profile PROF_DMVPN
 match identity remote address 0.0.0.0
 authentication local pre-share
 authentication remote pre-share
 keyring local KR_DMVPN
```

Los Spokes apuntan específicamente al Hub en su keyring, igual que en los labs anteriores.

### 2.5 Parámetros Criptográficos Utilizados

| Parámetro | Valor configurado | Propósito |
|---|---|---|
| **Cifrado IKE SA** | AES-256-CBC | Cifrado del canal IKE |
| **Integridad IKE SA** | SHA-256 | Integridad de mensajes IKE |
| **Grupo Diffie-Hellman** | Grupo 14 (2048-bit MODP) | Intercambio seguro de clave |
| **Autenticación** | Pre-Shared Key (PSK) | Autenticación mutua Hub-Spokes |
| **Lifetime IKE SA** | 86400 segundos | Canal IKE antes de renegociar |
| **Cifrado ESP** | AES-256 | Cifrado del tráfico de la nube DMVPN |
| **Autenticación ESP** | SHA-256 HMAC | Integridad del tráfico |
| **Modo IPSec** | Transport | mGRE ya encapsula |
| **Modo del túnel** | `gre multipoint` | Múltiples destinos desde una sola interfaz |
| **NHRP Network ID** | 1 | Identificador de la nube DMVPN |
| **Subred del túnel** | 10.25.37.0/24 | Espacio de la nube mGRE |
| **Enrutamiento** | OSPF Área 0, network broadcast | Propagación de rutas LAN |

> **Nota sobre PRF:** En IOSv/IOS 15.x el comando `prf` dentro de `crypto ikev2 proposal` no está disponible. IOS lo deriva automáticamente del integrity algorithm.

---

## 3. Documentación de la Red

### 3.1 Topología

Misma topología física del laboratorio DMVPN Fase 2 — R1 como Hub, R2 y R3 como Spokes, todos conectados a la nube `192.168.1.0/24`. La diferencia está en los comandos NHRP (`redirect` / `shortcut`) y en el protocolo de routing (OSPF en vez de EIGRP).

```
                                    [ INTERNET / NET ]
                                    192.168.1.0/24
                                          │
                  ┌───────────────────────┼───────────────────────┐
                  │ e0/0: 192.168.1.10    │ e0/0: 192.168.1.20    │ e0/0: 192.168.1.30
          ┌───────┴───────┐       ┌───────┴───────┐       ┌───────┴───────┐
          │  Router R1    │       │  Router R2    │       │  Router R3    │
          │     HUB       │       │   SPOKE 1     │       │   SPOKE 2     │
          │  Tunnel0:     │       │  Tunnel0:     │       │  Tunnel0:     │
          │  10.25.37.1   │       │  10.25.37.2   │       │  10.25.37.3   │
          │  OSPF DR      │       │  OSPF Pri=0   │       │  OSPF Pri=0   │
          │  nhrp redirect│       │  nhrp shortcut│       │  nhrp shortcut│
          └───────┬───────┘       └───────┬───────┘       └───────┬───────┘
                  │ e0/1: 20.25.37.1/26   │ e0/1: 20.25.37.65/26  │ e0/1: 20.25.37.129/26
                  │                       │                        │
          ┌───────┴───────┐       ┌───────┴───────┐       ┌───────┴───────┐
          │      SW1      │       │      SW2      │       │      SW3      │
          └───────┬───────┘       └───────┬───────┘       └───────┬───────┘
                  │                       │                        │
              ┌───┴──┐               ┌───┴──┐                ┌────┴─┐
              │ PC1  │               │ PC2  │                │ PC3  │
              │  .2  │               │ .66  │                │ .130 │
              └──────┘               └──────┘                └──────┘
           20.25.37.0/26          20.25.37.64/26          20.25.37.128/26

  ════════════════════════════════════════════════════════════════════════
  Flujo DMVPN Fase 3 — NHRP Redirect (shortcut switching):
    1. PC2 hace ping a PC3. R2 envía el paquete al Hub (R1)
       porque OSPF anunció esa ruta vía el Hub.
    2. El Hub reenvía el paquete a R3 Y envía un NHRP Redirect
       a R2: "para 20.25.37.128/26, ve directo a 10.25.37.3".
    3. R2 instala una ruta NHRP Shortcut (H) en su tabla.
    4. Los paquetes siguientes van R2 → R3 directo, cifrados
       por IKEv2/IPSec sin pasar por el Hub.
  ════════════════════════════════════════════════════════════════════════
```

### 3.2 Tabla de Dispositivos y Direccionamiento IP

| Dispositivo | Tipo / Modelo | Interfaz | Dirección IP | Máscara | Rol |
|---|---|---|---|---|---|
| **NET** | Cloud/Switch | — | 192.168.1.0/24 | /24 | Simulación de Internet (NBMA) |
| **R1** | Cisco IOS (Router) | e0/0 | 192.168.1.10 | /24 | WAN — Hub |
| | | e0/1 | 20.25.37.1 | /26 | Gateway LAN Hub |
| | | Tunnel0 | 10.25.37.1 | /24 | mGRE Hub / NHS / OSPF DR |
| **R2** | Cisco IOS (Router) | e0/0 | 192.168.1.20 | /24 | WAN — Spoke 1 |
| | | e0/1 | 20.25.37.65 | /26 | Gateway LAN Spoke 1 |
| | | Tunnel0 | 10.25.37.2 | /24 | mGRE Spoke 1 |
| **R3** | Cisco IOS (Router) | e0/0 | 192.168.1.30 | /24 | WAN — Spoke 2 |
| | | e0/1 | 20.25.37.129 | /26 | Gateway LAN Spoke 2 |
| | | Tunnel0 | 10.25.37.3 | /24 | mGRE Spoke 2 |
| **SW1** | Switch L2 | — | — | — | Conmutación LAN Hub |
| **SW2** | Switch L2 | — | — | — | Conmutación LAN Spoke 1 |
| **SW3** | Switch L2 | — | — | — | Conmutación LAN Spoke 2 |
| **PC1** | VPC | eth0 | 20.25.37.2 | /26 | Host Hub |
| **PC2** | VPC | eth0 | 20.25.37.66 | /26 | Host Spoke 1 |
| **PC3** | VPC | eth0 | 20.25.37.130 | /26 | Host Spoke 2 |

---

## 4. Scripts de Configuración

### 4.1 Router R1 — HUB

```cisco
! ══════════════════════════════════════════════════════
! R1 — HUB | DMVPN Fase 3 con IKEv2 + OSPF
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R1

! ─── Interfaces físicas ────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-NET
 ip address 192.168.1.10 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-HUB
 ip address 20.25.37.1 255.255.255.192
 no shutdown

! ══════════════════════════════════════════════════════
! BLOQUE IKEv2
! ══════════════════════════════════════════════════════

! ─── PASO 1: IKEv2 Proposal ────────────────────────────
! NOTA: no incluir "prf sha256" — no soportado en IOSv.
crypto ikev2 proposal PROP_DMVPN
 encryption aes-cbc-256
 integrity sha256
 group 14

! ─── PASO 2: IKEv2 Policy ──────────────────────────────
crypto ikev2 policy POL_DMVPN
 proposal PROP_DMVPN

! ─── PASO 3: IKEv2 Keyring — PSK wildcard para cualquier Spoke
crypto ikev2 keyring KR_DMVPN
 peer SPOKES_ANY
  address 0.0.0.0 0.0.0.0
  pre-shared-key local  DmvpnITLA2026!
  pre-shared-key remote DmvpnITLA2026!

! ─── PASO 4: IKEv2 Profile — acepta cualquier Spoke ────
crypto ikev2 profile PROF_DMVPN
 match identity remote address 0.0.0.0
 authentication local pre-share
 authentication remote pre-share
 keyring local KR_DMVPN

! ══════════════════════════════════════════════════════
! BLOQUE IPSEC
! ══════════════════════════════════════════════════════

! ─── PASO 5: Transform Set en modo TRANSPORTE ─────────
crypto ipsec transform-set TS_DMVPN esp-aes 256 esp-sha256-hmac
 mode transport

! ─── PASO 6: IPSec Profile con IKEv2 ──────────────────
crypto ipsec profile DMVPN_PROFILE
 set transform-set TS_DMVPN
 set ikev2-profile PROF_DMVPN

! ─── PASO 7: Interfaz mGRE del Hub ─────────────────────
interface Tunnel0
 description DMVPN-HUB-Fase3
 ip address 10.25.37.1 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 20250737
 ip nhrp network-id 1
 ip nhrp authentication dmvpnkey
 ip nhrp map multicast dynamic
 ip nhrp redirect              ! Fase 3 — Hub envía NHRP Redirect a los Spokes
 ip ospf network broadcast     ! Trata la nube mGRE como red broadcast
 ip ospf priority 2            ! Fuerza al Hub a ser siempre el OSPF DR
 tunnel protection ipsec profile DMVPN_PROFILE
 no shutdown

! ─── PASO 8: OSPF ──────────────────────────────────────
! Con OSPF broadcast no se necesita "no ip next-hop-self"
! ni "no ip split-horizon" — OSPF maneja esto correctamente.
router ospf 1
 network 20.25.37.0 0.0.0.63 area 0
 network 10.25.37.0 0.0.0.255 area 0
```

### 4.2 Router R2 — SPOKE 1

```cisco
! ══════════════════════════════════════════════════════
! R2 — SPOKE 1 | DMVPN Fase 3 con IKEv2 + OSPF
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R2

! ─── Interfaces físicas ────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-NET
 ip address 192.168.1.20 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-Spoke1
 ip address 20.25.37.65 255.255.255.192
 no shutdown

! ─── PASO 1: IKEv2 Proposal ────────────────────────────
crypto ikev2 proposal PROP_DMVPN
 encryption aes-cbc-256
 integrity sha256
 group 14

! ─── PASO 2: IKEv2 Policy ──────────────────────────────
crypto ikev2 policy POL_DMVPN
 proposal PROP_DMVPN

! ─── PASO 3: IKEv2 Keyring — apunta específicamente al Hub
crypto ikev2 keyring KR_DMVPN
 peer HUB_R1
  address 192.168.1.10
  pre-shared-key local  DmvpnITLA2026!
  pre-shared-key remote DmvpnITLA2026!

! ─── PASO 4: IKEv2 Profile ─────────────────────────────
crypto ikev2 profile PROF_DMVPN
 match identity remote address 192.168.1.10 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KR_DMVPN

! ─── PASO 5: Transform Set en modo TRANSPORTE ─────────
crypto ipsec transform-set TS_DMVPN esp-aes 256 esp-sha256-hmac
 mode transport

! ─── PASO 6: IPSec Profile con IKEv2 ──────────────────
crypto ipsec profile DMVPN_PROFILE
 set transform-set TS_DMVPN
 set ikev2-profile PROF_DMVPN

! ─── PASO 7: Interfaz mGRE del Spoke ───────────────────
interface Tunnel0
 description DMVPN-Spoke1-Fase3
 ip address 10.25.37.2 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 20250737
 ip nhrp network-id 1
 ip nhrp authentication dmvpnkey
 ip nhrp nhs 10.25.37.1
 ip nhrp map 10.25.37.1 192.168.1.10
 ip nhrp map multicast 192.168.1.10
 ip nhrp shortcut              ! Fase 3 — instala rutas NHRP Shortcut (H) en la tabla
 ip ospf network broadcast
 ip ospf priority 0            ! Excluye al Spoke de la elección DR/BDR
 tunnel protection ipsec profile DMVPN_PROFILE
 no shutdown

! ─── PASO 8: OSPF ──────────────────────────────────────
router ospf 1
 network 20.25.37.64 0.0.0.63 area 0
 network 10.25.37.0 0.0.0.255 area 0
```

### 4.3 Router R3 — SPOKE 2

```cisco
! ══════════════════════════════════════════════════════
! R3 — SPOKE 2 | DMVPN Fase 3 con IKEv2 + OSPF
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R3

! ─── Interfaces físicas ────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-NET
 ip address 192.168.1.30 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-Spoke2
 ip address 20.25.37.129 255.255.255.192
 no shutdown

! ─── PASO 1: IKEv2 Proposal ────────────────────────────
crypto ikev2 proposal PROP_DMVPN
 encryption aes-cbc-256
 integrity sha256
 group 14

! ─── PASO 2: IKEv2 Policy ──────────────────────────────
crypto ikev2 policy POL_DMVPN
 proposal PROP_DMVPN

! ─── PASO 3: IKEv2 Keyring ─────────────────────────────
crypto ikev2 keyring KR_DMVPN
 peer HUB_R1
  address 192.168.1.10
  pre-shared-key local  DmvpnITLA2026!
  pre-shared-key remote DmvpnITLA2026!

! ─── PASO 4: IKEv2 Profile ─────────────────────────────
crypto ikev2 profile PROF_DMVPN
 match identity remote address 192.168.1.10 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KR_DMVPN

! ─── PASO 5: Transform Set en modo TRANSPORTE ─────────
crypto ipsec transform-set TS_DMVPN esp-aes 256 esp-sha256-hmac
 mode transport

! ─── PASO 6: IPSec Profile con IKEv2 ──────────────────
crypto ipsec profile DMVPN_PROFILE
 set transform-set TS_DMVPN
 set ikev2-profile PROF_DMVPN

! ─── PASO 7: Interfaz mGRE del Spoke ───────────────────
interface Tunnel0
 description DMVPN-Spoke2-Fase3
 ip address 10.25.37.3 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 20250737
 ip nhrp network-id 1
 ip nhrp authentication dmvpnkey
 ip nhrp nhs 10.25.37.1
 ip nhrp map 10.25.37.1 192.168.1.10
 ip nhrp map multicast 192.168.1.10
 ip nhrp shortcut
 ip ospf network broadcast
 ip ospf priority 0
 tunnel protection ipsec profile DMVPN_PROFILE
 no shutdown

! ─── PASO 8: OSPF ──────────────────────────────────────
router ospf 1
 network 20.25.37.128 0.0.0.63 area 0
 network 10.25.37.0 0.0.0.255 area 0
```

### 4.4 Configuración de PCs (Hosts)

**PC1 — LAN del Hub (`20.25.37.0/26`)**

```bash
ip 20.25.37.2 255.255.255.192 20.25.37.1
```

**PC2 — LAN de Spoke 1 (`20.25.37.64/26`)**

```bash
ip 20.25.37.66 255.255.255.192 20.25.37.65
```

**PC3 — LAN de Spoke 2 (`20.25.37.128/26`)**

```bash
ip 20.25.37.130 255.255.255.192 20.25.37.129
```

---

## 5. Verificación del Túnel

### 5.1 Verificar el registro DMVPN en el Hub

```cisco
R1# show dmvpn
```

*Salida esperada:*

```
Interface: Tunnel0, IPv4 NHRP Details
Type:Hub, NHRP Peers:2,

 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1 192.168.1.20         10.25.37.2    UP 00:02:15     D
     1 192.168.1.30         10.25.37.3    UP 00:01:48     D
```

---

### 5.2 Verificar la IKEv2 SA

```cisco
R1# show crypto ikev2 sa
```

*Salida esperada:*

```
 IPv4 Crypto IKEv2  SA

Tunnel-id Local                 Remote                fvrf/ivrf      Status
1         192.168.1.10/500      192.168.1.20/500      none/none      READY
2         192.168.1.10/500      192.168.1.30/500      none/none      READY
```

> Dos SAs `READY` — una por cada Spoke registrado.

---

### 5.3 Verificar OSPF DR/BDR en el Hub

```cisco
R1# show ip ospf neighbor
```

*Salida esperada:*

```
Neighbor ID     Pri   State           Dead Time   Address         Interface
10.25.37.2        0   FULL/DROTHER    00:00:35    10.25.37.2      Tunnel0
10.25.37.3        0   FULL/DROTHER    00:00:32    10.25.37.3      Tunnel0
```

> Los Spokes aparecen como `DROTHER` (priority 0 — no pueden ser DR). El Hub (R1) es el DR con priority 2. Estado `FULL` confirma adyacencia OSPF completa.

---

### 5.4 Verificar rutas OSPF en un Spoke

```cisco
R2# show ip route ospf
```

*Salida antes del ping Spoke-Spoke — rutas vía Hub:*

```
O     20.25.37.0/26  [110/1001] via 10.25.37.1, Tunnel0
O     20.25.37.128/26 [110/1002] via 10.25.37.1, Tunnel0
```

---

### 5.5 Disparar el NHRP Shortcut — ping Spoke-to-Spoke

```cisco
R2# ping 20.25.37.130 source 20.25.37.65 repeat 10
```

*Resultado esperado:*

```
Packet sent with a source address of 20.25.37.65
!!!!!!!!!! 
Success rate is 100 percent (10/10)
```

> Los primeros paquetes pueden ir vía Hub mientras se establece el Shortcut. Con `repeat 10` los últimos paquetes ya deben ir directo.

---

### 5.6 Verificar la ruta NHRP Shortcut instalada

Inmediatamente después del ping:

```cisco
R2# show ip route
```

*Buscar la entrada marcada con `H`:*

```
H     20.25.37.128/26 [250/1] via 10.25.37.3, 00:00:03, Tunnel0
```

> La `H` indica **NHRP Shortcut** — esta ruta fue instalada por el mecanismo de Fase 3 (NHRP Redirect del Hub). El next-hop es `10.25.37.3` (directo a R3), no `10.25.37.1` (Hub). Esta entrada es la **prueba definitiva de que DMVPN Fase 3 está funcionando**.

---

### 5.7 Verificar la caché NHRP en el Spoke

```cisco
R2# show ip nhrp
```

*Salida esperada post-ping:*

```
10.25.37.1/32 via 10.25.37.1
   Tunnel0 created HH:MM:SS, never expire
   Type: static, Flags: used
   NBMA address: 192.168.1.10
10.25.37.3/32 via 10.25.37.3
   Tunnel0 created 00:00:05, expire 00:02:55
   Type: dynamic, Flags: router used nhop
   NBMA address: 192.168.1.30
```

---

### 5.8 Tabla de comandos de verificación

| Comando | Qué muestra |
|---|---|
| `show dmvpn` | Peers NHRP registrados en la nube, estado UP/DOWN, tipo D (dynamic). |
| `show crypto ikev2 sa` | SAs IKEv2 activas — debe mostrar `READY` para cada Spoke. |
| `show ip ospf neighbor` | Adyacencias OSPF — Spokes deben aparecer como `FULL/DROTHER`. |
| `show ip route ospf` | Rutas OSPF antes del shortcut — next-hop vía Hub. |
| `show ip route` | Buscar entradas `H` (NHRP Shortcut) post-ping — prueba de Fase 3. |
| `show ip nhrp` | Caché NHRP — entrada dinámica con NBMA `192.168.1.30` post-ping. |
| `show crypto ipsec sa` | Child SAs activas con contadores de paquetes. |
| `debug nhrp` | Debug del proceso de Redirect/Shortcut en tiempo real. |

---

## 6. Capturas de Pantalla

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, los tres routers encendidos. |
| 2 | [`02_config_hub_fase3.png`](screenshots/02_config_hub_fase3.png) | Consola de R1 mostrando `ip nhrp redirect`, `ip ospf network broadcast`, `ip ospf priority 2` y `set ikev2-profile` en el IPSec Profile. |
| 3 | [`03_config_spoke_fase3.png`](screenshots/03_config_spoke_fase3.png) | Consola de R2 mostrando `ip nhrp shortcut`, `ip ospf network broadcast` e `ip ospf priority 0`. |
| 4 | [`04_show_dmvpn.png`](screenshots/04_show_dmvpn.png) | Salida de `show dmvpn` en R1 mostrando ambos Spokes con estado `UP` y `Attrb: D`. |
| 5 | [`05_ikev2_sa_ready.png`](screenshots/05_ikev2_sa_ready.png) | Salida de `show crypto ikev2 sa` en R1 mostrando dos SAs en estado `READY`. |
| 6 | [`06_ospf_neighbors.png`](screenshots/06_ospf_neighbors.png) | Salida de `show ip ospf neighbor` en R1 mostrando los Spokes como `FULL/DROTHER`. |
| 7 | [`07_ping_spoke_spoke.png`](screenshots/07_ping_spoke_spoke.png) | Ping exitoso desde R2 (`source 20.25.37.65`) hacia PC3 (`20.25.37.130`) con `repeat 10`. |

---

## 7. Consideraciones de Seguridad

### 7.1 PSK wildcard en el Hub con IKEv2

La PSK wildcard (`address 0.0.0.0 0.0.0.0` en el keyring) es necesaria para que el Hub acepte cualquier Spoke dinámicamente. En producción esto se mitiga con:

* Certificados PKI — cada Spoke tiene su propio certificado firmado por la CA corporativa.
* ACL en la interfaz WAN del Hub limitando qué rangos de IP pueden iniciar IKEv2.

### 7.2 Diferencia de comandos Fase 2 vs. Fase 3

```cisco
! Fase 2 — sin comandos de redirect/shortcut:
interface Tunnel0
 ip nhrp map multicast dynamic
 ! sin ip nhrp redirect
 ! sin ip nhrp shortcut en los Spokes

! Fase 3 — agrega exactamente dos comandos:
! En el Hub:
 ip nhrp redirect       ! Hub envía NHRP Redirect a los Spokes

! En cada Spoke:
 ip nhrp shortcut       ! Spoke acepta e instala rutas H en su tabla
```

Esos dos comandos son la única diferencia de configuración entre Fase 2 y Fase 3.

### 7.3 Hardening recomendado

```cisco
! Deshabilitar IKEv1 (solo usamos IKEv2)
no crypto isakmp enable

! Forzar PFS en el IPSec Profile
crypto ipsec profile DMVPN_PROFILE
 set pfs group14

! Limitar registros NHRP (anti-DoS)
interface Tunnel0
 ip nhrp max-send 100 every 10
```

---

## 8. Video Demostrativo

🎥 **[Ver demostración en YouTube](https://youtu.be/WLUa1-bSpPY)**

**Duración:** 7:20

**Contenido del video:**

* ✅ Topología funcional en PNETLab con nombre completo `Jordy Rosario — 20250737` visible.
* ✅ Reloj del sistema operativo visible evidenciando fecha y hora actual.
* ✅ Rostro y voz del autor realizando la explicación técnica del laboratorio.
* ✅ Configuración aplicada en R1 (Hub), R2 y R3 (Spokes).
* ✅ `show dmvpn` mostrando ambos Spokes registrados.
* ✅ `show crypto ikev2 sa` mostrando dos SAs `READY`.
* ✅ `show ip ospf neighbor` mostrando los Spokes como `FULL/DROTHER`.
* ✅ Ping Spoke-to-Spoke con `repeat 10`.

---

## 9. Referencias

* Cisco Systems. (2024). *Dynamic Multipoint VPN (DMVPN) Design Guide — Phase 3*.
* Cisco Systems. (2024). *Cisco IOS Security Configuration Guide — DMVPN with IKEv2*.
* Luciani, J. et al. (1998). *RFC 2332 — NBMA Next Hop Resolution Protocol (NHRP)*. IETF.
* Kaufman, C. et al. (2014). *RFC 7296 — Internet Key Exchange Protocol Version 2 (IKEv2)*. IETF.
* Hanks, S. et al. (1994). *RFC 1701 — Generic Routing Encapsulation (GRE)*. IETF.
* Doraswamy, N. & Harkins, D. (2003). *IPSec: The New Security Standard for the Internet, Intranets, and Virtual Private Networks (2nd Ed.)*. Prentice Hall.
