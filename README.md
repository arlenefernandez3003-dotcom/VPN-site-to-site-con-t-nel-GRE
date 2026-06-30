# 🔐 IPSec IKEv1 — VPN Site-to-Site con Túnel GRE

<div align="center">

**Instituto Tecnológico de las Américas — ITLA**  
Seguridad de Redes · Prof. Jonathan Esteban Rondón Corniel  
**Arlene Fernández Herrera · Matrícula: 2025-0730**

![IPSec](https://img.shields.io/badge/Protocolo-IPSec%20IKEv1-blue?style=for-the-badge&logo=cisco)
![Type](https://img.shields.io/badge/Tipo-Site--to--Site-brightgreen?style=for-the-badge)
![Mode](https://img.shields.io/badge/Modo-GRE%20over%20IPSec-orange?style=for-the-badge)
![Platform](https://img.shields.io/badge/Plataforma-PNetLab-blueviolet?style=for-the-badge)

</div>

---

## 📑 Tabla de Contenidos

1. [Objetivo](#1-objetivo)
2. [Topología](#2-topología)
3. [Direccionamiento IP](#3-direccionamiento-ip)
4. [Parámetros Configurados](#4-parámetros-configurados)
5. [Scripts de Configuración](#5-scripts-de-configuración)
6. [Verificación del Túnel](#6-verificación-del-túnel)
7. [Capturas de Pantalla](#7-capturas-de-pantalla)
8. [Video Demostrativo](#8-video-demostrativo)

---

## 1. Objetivo

Implementar y verificar una **VPN Site-to-Site con túnel GRE protegido por IPSec IKEv1** en routers Cisco IOS dentro de PNetLab. La práctica cubre:

- Creación de un **túnel GRE** entre R1 y R2 como capa de encapsulamiento inicial, permitiendo el transporte de tráfico multicast y protocolos de enrutamiento dinámico.
- Protección del túnel GRE con **IPSec IKEv1**, cifrando el tráfico mediante Crypto Map aplicado sobre la interfaz WAN física.
- Comunicación cifrada entre `202.50.73.128/25` (Site A) y `202.50.73.0/25` (Site B) a través del segmento `192.168.19.0/24` de la nube NAT de PNetLab.
- Validación del túnel con comandos `show`, estado de la interfaz GRE y pruebas de conectividad extremo a extremo.

### ¿Por qué GRE over IPSec?

GRE por sí solo no cifra — solo encapsula. IPSec por sí solo (policy-based) no soporta multicast ni enrutamiento dinámico sobre el túnel. La combinación **GRE over IPSec** resuelve ambas limitaciones:

```
Paquete original (PC1 → PC2)
        │
        ▼
[Encapsulado en GRE]       ← GRE añade header, permite multicast/OSPF
        │
        ▼
[Cifrado con IPSec/ESP]    ← IPSec protege el paquete GRE completo
        │
        ▼
  Viaja por ISP cifrado → R2 descifra → desencapsula GRE → entrega a PC2
```

La ACL de tráfico interesante de IPSec apunta al protocolo GRE entre las IPs WAN de los peers, no a las LANs directamente.

---

## 2. Topología

```
                              [ INTERNET / ISP ]
                               192.168.19.0/24
                              (Cloud/NAT: 192.168.19.2)
                                      │
                  ┌───────────────────┴───────────────────┐
                  │ e0/0: 192.168.19.5                     │ e0/0: 192.168.19.6
          ┌───────┴────────┐                      ┌────────┴───────┐
          │   R1 (Peer A)  │◄══ Tunnel0 (GRE) ════►  R2 (Peer B)  │
          │                │   cifrado con IPSec   │               │
          └───────┬────────┘                       └────────┬──────┘
                  │ e0/1: 202.50.73.129/25                  │ e0/1: 202.50.73.1/25
                  │                                         │
          ┌───────┴────────┐                      ┌─────────┴──────┐
          │     SW1        │                      │      SW2       │
          └───────┬────────┘                      └────────┬───────┘
                  │                                        │
          ┌───────┴────────┐                      ┌────────┴───────┐
          │      PC1       │                      │      PC2       │
          │ 202.50.73.130  │                      │  202.50.73.2   │
          └────────────────┘                      └────────────────┘
           ◄── SITE A ──►                          ◄── SITE B ──►
           202.50.73.128/25                        202.50.73.0/25
```

> El túnel GRE (`Tunnel0`) se establece entre `192.168.19.5` y `192.168.19.6`.  
> El Crypto Map cifra con IPSec todo el tráfico GRE (protocolo IP 47) que fluye entre ambas IPs WAN.

**Flujo del tráfico:**
1. PC1 envía tráfico hacia PC2 → R1 lo enruta hacia `Tunnel0`.
2. GRE encapsula el paquete original añadiendo un nuevo header IP (src: `192.168.19.5`, dst: `192.168.19.6`).
3. La ACL de IPSec detecta ese paquete GRE como tráfico interesante y lo cifra con ESP.
4. El paquete cifrado viaja por el segmento ISP → R2 descifra → desencapsula GRE → entrega a PC2.

---

## 3. Direccionamiento IP

### Interfaces de Dispositivos

| Dispositivo | Interfaz    | Dirección IP       | Máscara | Gateway        | Rol                         |
|-------------|-------------|---------------------|---------|-----------------|------------------------------|
| Cloud (NAT) | —           | 192.168.19.2        | /24     | —               | Gateway Cloud NAT (PNetLab)  |
| **R1**      | **e0/0**    | **192.168.19.5**    | **/24** | 192.168.19.2    | WAN Peer A → Cloud           |
| **R1**      | **e0/1**    | **202.50.73.129**   | **/25** | —               | Gateway LAN Site A           |
| **R1**      | **Tunnel0** | **10.0.0.1**        | **/30** | —               | Túnel GRE hacia R2           |
| **R2**      | **e0/0**    | **192.168.19.6**    | **/24** | 192.168.19.2    | WAN Peer B → Cloud           |
| **R2**      | **e0/1**    | **202.50.73.1**     | **/25** | —               | Gateway LAN Site B           |
| **R2**      | **Tunnel0** | **10.0.0.2**        | **/30** | —               | Túnel GRE hacia R1           |
| SW1         | —           | —                   | —       | —               | Capa 2 Site A                |
| SW2         | —           | —                   | —       | —               | Capa 2 Site B                |
| PC1         | eth0        | 202.50.73.130       | /25     | 202.50.73.129   | Host Site A                  |
| PC2         | eth0        | 202.50.73.2         | /25     | 202.50.73.1     | Host Site B                  |

### Tabla de Subredes

| Subred              | Rango Utilizable               | Broadcast        | Uso                     |
|---------------------|----------------------------------|------------------|--------------------------|
| `192.168.19.0/24`   | 192.168.19.1 – 192.168.19.254   | 192.168.19.255   | Segmento WAN/Cloud       |
| `202.50.73.0/25`    | 202.50.73.1 – 202.50.73.126     | 202.50.73.127    | LAN Site B               |
| `202.50.73.128/25`  | 202.50.73.129 – 202.50.73.254   | 202.50.73.255    | LAN Site A               |
| `10.0.0.0/30`       | 10.0.0.1 – 10.0.0.2             | 10.0.0.3         | Direccionamiento Tunnel GRE |

---

## 4. Parámetros Configurados

### IKE Fase 1 — ISAKMP Policy

| Parámetro          | Valor               | Descripción                                              |
|---------------------|---------------------|-----------------------------------------------------------|
| Número de política | `10`                | Prioridad de la política ISAKMP                          |
| Cifrado            | AES-256             | Cifrado simétrico del canal IKE                          |
| Hash / Integridad  | SHA-256             | Verificación de integridad de mensajes IKE               |
| Autenticación      | Pre-Shared Key      | Clave compartida entre ambos peers                       |
| Grupo DH           | Group 14 (2048-bit) | Intercambio Diffie-Hellman para derivar clave de sesión  |
| Lifetime SA        | 86400 s (24 h)      | Duración del canal IKE antes de renegociar               |
| Pre-Shared Key     | `ITLA2025Arlene`    | Clave idéntica en R1 y R2                                |

### IKE Fase 2 — IPSec / Transform Set

| Parámetro      | Valor              | Descripción                                       |
|-----------------|--------------------|---------------------------------------------------|
| Transform Set  | `TS_AES256_SHA256` | Cifrado ESP-AES-256 + ESP-SHA256-HMAC             |
| Modo           | Tunnel             | Encapsulamiento del paquete GRE completo          |
| Lifetime SA    | 3600 s (1 h)       | Duración del túnel de datos antes de renegociar   |

### Tráfico Interesante (ACL para IPSec sobre GRE)

| Router | ACL                  | Fuente            | Destino           | Protocolo |
|--------|----------------------|-------------------|-------------------|-----------|
| R1     | `ACL_GRE_IPSEC`      | 192.168.19.5      | 192.168.19.6      | GRE (47)  |
| R2     | `ACL_GRE_IPSEC`      | 192.168.19.6      | 192.168.19.5      | GRE (47)  |

> La ACL **no** apunta a las LANs — apunta al tráfico GRE entre las IPs WAN. Todo lo que viaje dentro del túnel GRE queda automáticamente cifrado.

### Interfaz Tunnel GRE

| Parámetro          | R1            | R2            |
|---------------------|---------------|---------------|
| IP Tunnel           | 10.0.0.1/30   | 10.0.0.2/30   |
| Tunnel Source       | Ethernet0/0   | Ethernet0/0   |
| Tunnel Destination  | 192.168.19.6  | 192.168.19.5  |
| Tunnel Mode         | gre ip        | gre ip        |

---

## 5. Scripts de Configuración

### R1 — Site A

```cisco
! ══════════════════════════════════════════════════════════════
!  R1 — Site A | IPSec IKEv1 + GRE Tunnel
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

hostname R1

! ── Interfaces físicas ──────────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-Cloud
 ip address 192.168.19.5 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteA
 ip address 202.50.73.129 255.255.255.128
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.19.2

! ── Paso 1: ISAKMP Policy (IKE Fase 1) ──────────────────────
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key ITLA2025Arlene address 192.168.19.6

! ── Paso 2: Transform Set (IKE Fase 2) ──────────────────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ── Paso 3: ACL — tráfico interesante (protocolo GRE) ───────
ip access-list extended ACL_GRE_IPSEC
 permit gre host 192.168.19.5 host 192.168.19.6

! ── Paso 4: Crypto Map sobre interfaz WAN ───────────────────
crypto map CMAP_GRE 10 ipsec-isakmp
 set peer 192.168.19.6
 set transform-set TS_AES256_SHA256
 match address ACL_GRE_IPSEC
 set security-association lifetime seconds 3600

interface Ethernet0/0
 crypto map CMAP_GRE

! ── Paso 5: Interfaz de Túnel GRE ────────────────────────────
interface Tunnel0
 description Tunel-GRE-hacia-SiteB-R2
 ip address 10.0.0.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 192.168.19.6
 tunnel mode gre ip

! ── Paso 6: Ruta hacia LAN remota vía Tunnel0 ────────────────
ip route 202.50.73.0 255.255.255.128 Tunnel0
```

### R2 — Site B

```cisco
! ══════════════════════════════════════════════════════════════
!  R2 — Site B | IPSec IKEv1 + GRE Tunnel
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

hostname R2

! ── Interfaces físicas ──────────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-Cloud
 ip address 192.168.19.6 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteB
 ip address 202.50.73.1 255.255.255.128
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.19.2

! ── Paso 1: ISAKMP Policy (IKE Fase 1) ──────────────────────
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key ITLA2025Arlene address 192.168.19.5

! ── Paso 2: Transform Set (IKE Fase 2) ──────────────────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ── Paso 3: ACL — tráfico interesante (protocolo GRE) ───────
ip access-list extended ACL_GRE_IPSEC
 permit gre host 192.168.19.6 host 192.168.19.5

! ── Paso 4: Crypto Map sobre interfaz WAN ───────────────────
crypto map CMAP_GRE 10 ipsec-isakmp
 set peer 192.168.19.5
 set transform-set TS_AES256_SHA256
 match address ACL_GRE_IPSEC
 set security-association lifetime seconds 3600

interface Ethernet0/0
 crypto map CMAP_GRE

! ── Paso 5: Interfaz de Túnel GRE ────────────────────────────
interface Tunnel0
 description Tunel-GRE-hacia-SiteA-R1
 ip address 10.0.0.2 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 192.168.19.5
 tunnel mode gre ip

! ── Paso 6: Ruta hacia LAN remota vía Tunnel0 ────────────────
ip route 202.50.73.128 255.255.255.128 Tunnel0
```

### Hosts (VPCs en PNetLab)

```bash
# PC1 — Site A
ip 202.50.73.130 255.255.255.128 202.50.73.129

# PC2 — Site B
ip 202.50.73.2 255.255.255.128 202.50.73.1
```

---

## 6. Verificación del Túnel

### Estado de la interfaz Tunnel GRE

```cisco
R1# show interface tunnel 0
```

Salida esperada:

```
Tunnel0 is up, line protocol is up
  Internet address is 10.0.0.1/30
  Tunnel source 192.168.19.5, destination 192.168.19.6
  Tunnel protocol/transport GRE/IP
```

> `up/up` confirma que el túnel GRE está operativo. Si muestra `up/down`, revisar conectividad entre las IPs WAN (ping de `192.168.19.5` a `192.168.19.6`).

---

### Estado IKE Fase 1 — ISAKMP SA

```cisco
R1# show crypto isakmp sa
```

Salida esperada:

```
IPv4 Crypto ISAKMP SA
dst              src              state       conn-id  status
192.168.19.6     192.168.19.5     QM_IDLE     1001     ACTIVE
```

---

### Estado IPSec Fase 2 — IPSec SA

```cisco
R1# show crypto ipsec sa
```

Salida esperada (fragmento):

```
interface: Ethernet0/0
    Crypto map tag: CMAP_GRE, local addr 192.168.19.5

   local  ident: (192.168.19.5/255.255.255.255/47/0)
   remote ident: (192.168.19.6/255.255.255.255/47/0)

    #pkts encaps: 14, #pkts encrypt: 14, #pkts digest: 14
    #pkts decaps: 14, #pkts decrypt: 14, #pkts verify: 14
```

> El protocolo `47` corresponde a GRE — confirma que es el tráfico del túnel GRE lo que está siendo cifrado por IPSec.

---

### Conectividad extremo a extremo

```bash
# Desde PC1 hacia PC2
PC1> ping 202.50.73.2

# Desde R1 — ping al extremo del túnel GRE
R1# ping 10.0.0.2 source 10.0.0.1
```

Resultado esperado:

```
!!!!!!!!!!
Success rate is 100 percent (10/10)
```

---

### Tabla de Comandos de Verificación

| Comando | Qué verifica |
|---|---|
| `show interface tunnel 0` | Estado del túnel GRE (up/up indica operativo). |
| `show crypto isakmp sa` | Estado de IKE Fase 1. Debe mostrar `QM_IDLE`. |
| `show crypto ipsec sa` | SAs de Fase 2 con protocolo GRE (47) y contadores de paquetes. |
| `show ip route static` | Confirma ruta hacia LAN remota apuntando a `Tunnel0`. |
| `show crypto map` | Crypto Map aplicado en `Ethernet0/0` con ACL GRE. |

---
---

## 7. Capturas de Pantalla

| # | Captura | Descripción |
|---|---|---|
| 1 | [1.png](evidencias/1.png) | Topología en PNetLab con nombre y matrícula visibles, todos los nodos encendidos. |
| 2 | [2.png](evidencias/2.png) | Consola R1: configuración de `Tunnel0` (GRE) y Crypto Map con ACL de protocolo GRE. |
| 3 | [3.png](evidencias/3.png) | Salida de `show interface tunnel 0` (up/up) y `show crypto isakmp sa` (`QM_IDLE`). |
| 4 | [4.png](evidencias/4.png) | Ping exitoso de PC1 (`202.50.73.130`) a PC2 (`202.50.73.2`) con túnel GRE+IPSec activo. |

---

## 8. Video Demostrativo

🎥 **[Ver en YouTube — enlace pendiente](#)**

---

<div align="center">

**Arlene Fernández Herrera · 2025-0730 · ITLA**  
*Seguridad de Redes — Lab 03: IPSec IKEv1 GRE Tunnel VPN*

</div>
