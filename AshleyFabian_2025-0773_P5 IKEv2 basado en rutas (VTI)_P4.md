# VPN IPSec IKEv2 Site-to-Site Basado en Rutas (VTI)

**Nombre:** Ashley Fabian Ortiz  
**Matrícula:** 2025-0773  
**Práctica:** P4  

---

## Objetivo

Configurar un túnel VPN IPSec IKEv2 Site-to-Site basado en rutas mediante una **Virtual Tunnel Interface (VTI)** entre dos routers Cisco CSR1000v, permitiendo la comunicación cifrada entre dos redes LAN a través de un router ISP. Esta práctica combina la sintaxis de IKEv2 (vista en P4) con la arquitectura de túnel VTI (vista en P2), representando la evolución natural de ambas tecnologías hacia una solución más moderna, eficiente y escalable.

La diferencia fundamental respecto a P2 (IKEv1 VTI) es que el `crypto ipsec profile` ahora referencia un `ikev2-profile` explícito en vez de usar IKEv1 implícitamente. El resto de la arquitectura es idéntica: una interfaz Tunnel0 de tipo `ipsec ipv4` enruta todo el tráfico que atraviesa el túnel, cifrándolo automáticamente sin necesidad de ACLs.

### Comparación de las 6 prácticas Site-to-Site

| Práctica | Protocolo IKE | Método         | ACL necesaria | Multicast |
|----------|---------------|----------------|---------------|-----------|
| P1       | IKEv1         | Policy-Based   | Sí            | No        |
| P2       | IKEv1         | VTI            | No            | No        |
| P3       | IKEv1         | GRE over IPSec | Sí (GRE/47)   | Sí        |
| P4       | IKEv2         | Policy-Based   | Sí            | No        |
| P5       | IKEv2         | VTI            | No            | No        |
| P6       | IKEv2         | GRE over IPSec | Sí (GRE/47)   | Sí        |

---

## Topología

```
[PC1]--[SW-A]--[R1]--------[ISP]--------[R2]--[SW-B]--[PC2]
        LAN-A   Gi3    Gi1       Gi2    Gi2    LAN-B
              25.7.73.137  25.7.73.129  25.7.73.133  25.7.73.134  25.7.73.145

                       Tunnel0 (VTI)         Tunnel0 (VTI)
                    25.7.73.153 <----------> 25.7.73.154
                         IPSec IKEv2 (profile PROF-IPSEC-P5)
```

---

## Direccionamiento IP

| Dispositivo | Interfaz         | Dirección IP  | Máscara         | Descripción        |
|-------------|------------------|---------------|-----------------|--------------------|
| R1          | GigabitEthernet1 | 25.7.73.129   | 255.255.255.252 | WAN R1 → ISP       |
| ISP         | GigabitEthernet1 | 25.7.73.130   | 255.255.255.252 | WAN ISP → R1       |
| ISP         | GigabitEthernet2 | 25.7.73.133   | 255.255.255.252 | WAN ISP → R2       |
| R2          | GigabitEthernet2 | 25.7.73.134   | 255.255.255.252 | WAN R2 → ISP       |
| R1          | GigabitEthernet3 | 25.7.73.137   | 255.255.255.248 | Gateway LAN-A      |
| PC1         | eth0             | 25.7.73.138   | 255.255.255.248 | Host LAN-A         |
| R2          | GigabitEthernet3 | 25.7.73.145   | 255.255.255.248 | Gateway LAN-B      |
| PC2         | eth0             | 25.7.73.146   | 255.255.255.248 | Host LAN-B         |
| R1          | Tunnel0 (VTI)    | 25.7.73.153   | 255.255.255.252 | Extremo VTI R1     |
| R2          | Tunnel0 (VTI)    | 25.7.73.154   | 255.255.255.252 | Extremo VTI R2     |

### Subredes utilizadas (bloque 25.7.73.128/27)

| Subred           | Rango utilizable            | Uso            |
|-------------------|-----------------------------|----------------|
| 25.7.73.128/30    | 25.7.73.129 – 25.7.73.130   | WAN R1 – ISP   |
| 25.7.73.132/30    | 25.7.73.133 – 25.7.73.134   | WAN ISP – R2   |
| 25.7.73.136/29    | 25.7.73.137 – 25.7.73.142   | LAN-A          |
| 25.7.73.144/29    | 25.7.73.145 – 25.7.73.150   | LAN-B          |
| 25.7.73.152/30    | 25.7.73.153 – 25.7.73.154   | Túnel VTI      |

---

## Parámetros de configuración IPSec IKEv2

### IKEv2 Proposal

| Parámetro  | Valor               |
|------------|---------------------|
| Cifrado    | AES-CBC 256         |
| Integridad | SHA-256             |
| Grupo DH   | Group 14 (2048-bit) |

### IKEv2 Policy

| Parámetro          | Valor    |
|--------------------|----------|
| Nombre             | POL-P5   |
| Proposal asociado  | PROP-P5  |

### IKEv2 Keyring

| Parámetro       | R1                | R2                |
|-----------------|-------------------|-------------------|
| Peer remoto     | R2 (25.7.73.134)  | R1 (25.7.73.129)  |
| Pre-shared key  | Cisco123!         | Cisco123!         |

### IKEv2 Profile

| Parámetro              | R1                | R2                |
|------------------------|-------------------|-------------------|
| Match identity remote  | 25.7.73.134/32    | 25.7.73.129/32    |
| Autenticación local    | pre-share         | pre-share         |
| Autenticación remota   | pre-share         | pre-share         |
| Keyring asociado       | KEY-P5            | KEY-P5            |

### IPSec Profile (diferencia clave vs P2)

| Parámetro          | Valor           |
|--------------------|-----------------|
| Nombre             | PROF-IPSEC-P5   |
| Transform Set      | TS-P5           |
| IKEv2 Profile      | PROF-P5         |

> En P2 (IKEv1 VTI) el `crypto ipsec profile` solo referenciaba el transform set. En P5 (IKEv2 VTI) se agrega `set ikev2-profile PROF-P5` para forzar el uso de IKEv2 en la negociación.

### Interfaz Tunnel0 (VTI)

| Parámetro          | R1              | R2              |
|--------------------|-----------------|-----------------|
| IP Tunnel          | 25.7.73.153/30  | 25.7.73.154/30  |
| Tunnel source      | Gi1             | Gi2             |
| Tunnel destination | 25.7.73.134     | 25.7.73.129     |
| Tunnel mode        | ipsec ipv4      | ipsec ipv4      |
| Tunnel protection  | PROF-IPSEC-P5   | PROF-IPSEC-P5   |

### Fase 2 — IPSec Transform Set

| Parámetro      | Valor        |
|----------------|--------------|
| Cifrado ESP    | AES 256      |
| Integridad ESP | SHA-256 HMAC |
| Modo           | Tunnel       |

---

## Scripts de configuración

### ISP

```
enable
configure terminal
hostname ISP

interface GigabitEthernet1
 ip address 25.7.73.130 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet2
 ip address 25.7.73.133 255.255.255.252
 no shutdown
 exit

end
write memory
```

### R1

```
enable
configure terminal
hostname R1

interface GigabitEthernet1
 ip address 25.7.73.129 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet3
 ip address 25.7.73.137 255.255.255.248
 no shutdown
 exit

ip route 0.0.0.0 0.0.0.0 25.7.73.130

! --- IKEv2 Proposal ---
crypto ikev2 proposal PROP-P5
 encryption aes-cbc-256
 integrity sha256
 group 14
 exit

! --- IKEv2 Policy ---
crypto ikev2 policy POL-P5
 proposal PROP-P5
 exit

! --- Keyring ---
crypto ikev2 keyring KEY-P5
 peer R2
  address 25.7.73.134
  pre-shared-key Cisco123!
  exit
 exit

! --- IKEv2 Profile ---
crypto ikev2 profile PROF-P5
 match identity remote address 25.7.73.134 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KEY-P5
 exit

! --- Transform Set ---
crypto ipsec transform-set TS-P5 esp-aes 256 esp-sha256-hmac
 mode tunnel
 exit

! --- IPSec Profile referenciando IKEv2 (diferencia clave vs P2) ---
crypto ipsec profile PROF-IPSEC-P5
 set transform-set TS-P5
 set ikev2-profile PROF-P5
 exit

! --- Interfaz Tunnel VTI ---
interface Tunnel0
 ip address 25.7.73.153 255.255.255.252
 tunnel source GigabitEthernet1
 tunnel destination 25.7.73.134
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF-IPSEC-P5
 no shutdown
 exit

! --- Ruta hacia LAN-B a través del túnel ---
ip route 25.7.73.144 255.255.255.248 25.7.73.154

end
write memory
```

### R2

```
enable
configure terminal
hostname R2

interface GigabitEthernet2
 ip address 25.7.73.134 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet3
 ip address 25.7.73.145 255.255.255.248
 no shutdown
 exit

ip route 0.0.0.0 0.0.0.0 25.7.73.133

! --- IKEv2 Proposal ---
crypto ikev2 proposal PROP-P5
 encryption aes-cbc-256
 integrity sha256
 group 14
 exit

! --- IKEv2 Policy ---
crypto ikev2 policy POL-P5
 proposal PROP-P5
 exit

! --- Keyring ---
crypto ikev2 keyring KEY-P5
 peer R1
  address 25.7.73.129
  pre-shared-key Cisco123!
  exit
 exit

! --- IKEv2 Profile ---
crypto ikev2 profile PROF-P5
 match identity remote address 25.7.73.129 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KEY-P5
 exit

! --- Transform Set ---
crypto ipsec transform-set TS-P5 esp-aes 256 esp-sha256-hmac
 mode tunnel
 exit

! --- IPSec Profile referenciando IKEv2 ---
crypto ipsec profile PROF-IPSEC-P5
 set transform-set TS-P5
 set ikev2-profile PROF-P5
 exit

! --- Interfaz Tunnel VTI ---
interface Tunnel0
 ip address 25.7.73.154 255.255.255.252
 tunnel source GigabitEthernet2
 tunnel destination 25.7.73.129
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF-IPSEC-P5
 no shutdown
 exit

! --- Ruta hacia LAN-A a través del túnel ---
ip route 25.7.73.136 255.255.255.248 25.7.73.153

end
write memory
```

### SW-A y SW-B (IOSvL2)

```
enable
configure terminal

interface GigabitEthernet0/0
 switchport mode access
 switchport access vlan 1
 no shutdown
 exit

interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 1
 no shutdown
 exit

end
write memory
```

### PC1 (VPCS)

```
ip 25.7.73.138 255.255.255.248 25.7.73.137
save
```

### PC2 (VPCS)

```
ip 25.7.73.146 255.255.255.248 25.7.73.145
save
```

---

## Verificación y pruebas

> Comandos ejecutados en R1 y R2. El ISP no participa de la VPN; los PCs solo ejecutan `ping`.

### Comandos de verificación

```
show interface Tunnel0
show crypto ikev2 sa
show crypto ipsec sa
show ip route
```

### Resultados obtenidos — R1

**show interface Tunnel0:**
```
Tunnel0 is up, line protocol is up
  Internet address is 25.7.73.153/30
  Tunnel protocol/transport IPSEC/IP
  Tunnel source 25.7.73.129 (GigabitEthernet1), destination 25.7.73.134
  Tunnel protection via IPSec (profile "PROF-IPSEC-P5")
```

**show crypto ikev2 sa:**
```
Tunnel-id Local                 Remote                fvrf/ivrf            Status
2         25.7.73.129/500       25.7.73.134/500       none/none            READY
      Encr: AES-CBC, keysize: 256, PRF: SHA256, Hash: SHA256, DH Grp:14, Auth sign: PSK, Auth verify: PSK
      Life/Active Time: 86400/616 sec
```

**show crypto ipsec sa (fragmento):**
```
local  ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
remote ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
in use settings ={Tunnel, }
Status: ACTIVE(ACTIVE)
```

**show ip route (fragmento):**
```
S     25.7.73.144/29 [1/0] via 25.7.73.154
C     25.7.73.152/30 is directly connected, Tunnel0
L     25.7.73.153/32 is directly connected, Tunnel0
```

### Pruebas de conectividad

**Ping PC1 → PC2:**
```
PC1> ping 25.7.73.146
25.7.73.146 icmp_seq=1 timeout
84 bytes from 25.7.73.146 icmp_seq=2 ttl=62 time=73.824 ms
84 bytes from 25.7.73.146 icmp_seq=3 ttl=62 time=67.862 ms
84 bytes from 25.7.73.146 icmp_seq=4 ttl=62 time=72.613 ms
```

**Ping PC2 → PC1:**
```
PC2> ping 25.7.73.138
84 bytes from 25.7.73.138 icmp_seq=1 ttl=62 time=139.594 ms
84 bytes from 25.7.73.138 icmp_seq=2 ttl=62 time=82.685 ms
84 bytes from 25.7.73.138 icmp_seq=3 ttl=62 time=48.124 ms
```

> El primer paquete de PC1 dio timeout por la negociación inicial de IKEv2 (IKE_SA_INIT + IKE_AUTH). Una vez establecida la SA, los paquetes subsiguientes viajan cifrados sin interrupción. Este comportamiento es idéntico al observado en P4 y es completamente normal en VPNs basadas en políticas o VTI con negociación on-demand.

---

## Capturas de pantalla

> Insertar aquí las capturas de:
> 1. Topología completa en GNS3 con nombre y matrícula visible
![](./Topologia.png)

> 2. `show interface Tunnel0` en R1 y R2 (IPSEC/IP, up/up, profile PROF-IPSEC-P5)
![](./C1R1.png)
![](./C1R2.png)

> 3. `show crypto ikev2 sa` en R1 y R2 (Status READY)
![](./C2R1.png)
![](./C2R2.png)

> 4. `show crypto ipsec sa` en R1 y R2 (Status ACTIVE, identifiers 0.0.0.0/0.0.0.0)
![](./C3R1.png)
![](./C3R2.png)

> 5. `show ip route` en R1 y R2 (ruta estática via Tunnel0)
![](./C4R1.png)
![](./C4R2.png)

> 6. Ping PC1 → PC2 (mostrando primer timeout y éxitos posteriores)
![](./PING1.png)

> 7. Ping PC2 → PC1 (exitoso desde el primer paquete)
![](./PING2.png)
---

## Conclusión

Se configuró exitosamente una VPN IPSec IKEv2 Site-to-Site basada en rutas (VTI) entre R1 y R2. La combinación de IKEv2 con VTI representa la arquitectura más moderna y recomendada para VPNs Site-to-Site en entornos Cisco IOS-XE: IKEv2 reduce los intercambios de negociación a 4 mensajes (vs 6-9 de IKEv1 Main Mode), mientras que VTI elimina la necesidad de ACLs para definir tráfico interesante, simplificando la configuración y facilitando la integración con protocolos de enrutamiento dinámico. La SA IKEv2 alcanzó el estado READY con AES-256, SHA-256 y DH Group 14, y la conectividad bidireccional entre LAN-A (25.7.73.136/29) y LAN-B (25.7.73.144/29) fue verificada exitosamente.
