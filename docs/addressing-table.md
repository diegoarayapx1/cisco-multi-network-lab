# Tabla de direccionamiento

## Direccionamiento IPv4

| Dispositivo | Interfaz | IP / Máscara |
|---|---|---|
| ISP-BGP-SA100 | Loopback0 | 8.8.8.8/32 |
| ISP-BGP-SA100 | Ethernet0/0 | 200.10.10.1/30 |
| ISP-BGP-SA100 | Ethernet0/1 | 200.20.20.1/30 |
| BR1 | Ethernet0/0 | 200.10.10.2/30 |
| BR1 | Ethernet0/1 | 192.168.50.1/30 |
| BR2 | Ethernet0/0 | 200.20.20.2/30 |
| BR2 | Ethernet0/2 | 172.16.50.1/30 |
| CORE_R1 | Ethernet0/0 | 192.168.50.2/30 |
| CORE_R1 | Ethernet0/2 | 192.168.51.1/30 |
| CORE_R1 | Ethernet0/3 | 192.168.52.1/30 |
| CORE_R2 | Ethernet0/1 | 172.16.50.2/30 |
| CORE_R2 | Ethernet0/2 | 172.16.51.1/30 |
| CORE_R2 | Ethernet0/3 | 172.16.52.1/30 |
| Dist_SWA | Ethernet0/0 | 192.168.51.2/30 |
| Dist_SWA | SVI Vlan20 | 192.168.20.2/24 |
| Dist_SWA | SVI Vlan21 | 192.168.21.2/24 |
| Dist_SWA | SVI Vlan22 | 192.168.22.2/24 |
| Dist_SWB | Ethernet0/0 | 192.168.52.2/30 |
| Dist_SWB | SVI Vlan20 | 192.168.20.3/24 |
| Dist_SWB | SVI Vlan21 | 192.168.21.3/24 |
| Dist_SWB | SVI Vlan22 | 192.168.22.3/24 |
| Host_1 | Ethernet0/0 | 172.16.51.2/30 (GW 172.16.51.1) |
| Host_2 | Ethernet0/0 | 172.16.52.2/30 (GW 172.16.52.1) |
| PC_VLAN20 | Ethernet0/0 | 192.168.20.10/24 (GW 192.168.20.1) |
| PC_VLAN21 | Ethernet0/0 | 192.168.21.10/24 (GW 192.168.21.1) |

## Segmentos VLAN (lado izquierdo, AS200)

| VLAN | Nombre | Red IPv4 | Red IPv6 |
|---|---|---|---|
| VLAN20 | RRHH | 192.168.20.0/24 | 3001:ABCD:ABCD:20::/64 |
| VLAN21 | Ventas | 192.168.21.0/24 | 3001:ABCD:ABCD:21::/64 |
| VLAN22 | Admin | 192.168.22.0/24 | 3001:ABCD:ABCD:22::/64 |

## HSRP IPv4

| VLAN | Grupo | VIP | Active | Standby |
|---|---|---|---|---|
| VLAN20 | 20 | 192.168.20.1 | Dist_SWA (pri 110) | Dist_SWB (pri 100) |
| VLAN21 | 21 | 192.168.21.1 | Dist_SWA (pri 110) | Dist_SWB (pri 100) |
| VLAN22 | 22 | 192.168.22.1 | Dist_SWA (pri 110) | Dist_SWB (pri 100) |

## HSRP IPv6

Debido a una limitación de IOS 15.2 (no permite combinar grupos HSRP IPv4 e IPv6 bajo el mismo número de grupo en la misma interfaz), se usaron grupos separados para IPv6 con autoconfig:

| VLAN | Grupo IPv6 | Active | Standby |
|---|---|---|---|
| VLAN20 | 120 | Dist_SWA (pri 110) | Dist_SWB (pri 100) |
| VLAN21 | 121 | Dist_SWA (pri 110) | Dist_SWB (pri 100) |
| VLAN22 | 122 | Dist_SWA (pri 110) | Dist_SWB (pri 100) |

## EtherChannel (LACP, modo active)

Mapeo de interfaces físicas a Port-Channel, verificado contra el netmap de IOU:

| Port-Channel | Extremo A | Extremo B | Función |
|---|---|---|---|
| Po1 | Dist_SWA Et1/0, Et1/1 | Dist_SWB Et1/0, Et1/1 | Enlace inter-distribución (HSRP) |
| Po2 | Dist_SWA Et0/1, Et0/2 | Acceso_SW1 Et0/0, Et0/1 | Distribución A ↔ Acceso |
| Po3 | Dist_SWB Et0/1, Et0/2 | Acceso_SW1 Et0/2, Et0/3 | Distribución B ↔ Acceso |

Puertos de acceso en Acceso_SW1: Et1/0 → PC_VLAN20 (VLAN20), Et1/1 → PC_VLAN21 (VLAN21), ambos con port-security sticky.

## Direccionamiento IPv6

| Segmento | Prefijo |
|---|---|
| ISP Loopback0 | 3001:ABCD:ABCD:8::8/128 |
| ISP↔BR1 | 3001:ABCD:ABCD:101::/126 |
| ISP↔BR2 | 3001:ABCD:ABCD:101::4/126 |
| BR1↔CORE_R1 | 3001:ABCD:ABCD:110::/64 |
| CORE_R1↔Dist_SWA | 3001:ABCD:ABCD:111::/64 |
| CORE_R1↔Dist_SWB | 3001:ABCD:ABCD:112::/64 |
| BR2↔CORE_R2 | 3001:ABCD:ABCD:120::/64 |
| CORE_R2↔Host_1 | 3001:ABCD:ABCD:121::/64 |
| CORE_R2↔Host_2 | 3001:ABCD:ABCD:122::/64 |
| VLAN20 | 3001:ABCD:ABCD:20::/64 |
| VLAN21 | 3001:ABCD:ABCD:21::/64 |
| VLAN22 | 3001:ABCD:ABCD:22::/64 |

## Sistemas autónomos BGP

| AS | Dispositivo | Rol |
|---|---|---|
| AS100 | ISP-BGP-SA100 | Tránsito entre AS200 y AS300 |
| AS200 | BR1 | Borde de la organización izquierda (Core/Dist/Acceso) |
| AS300 | BR2 | Borde de la organización derecha (topología simplificada) |

## VPN IPSec Site-to-Site

| Parámetro | Valor |
|---|---|
| Peer BR1 | 200.10.10.2 |
| Peer BR2 | 200.20.20.2 |
| Pre-shared key | CISCO |
| Transform-set | esp-aes esp-sha-hmac, modo túnel |
| ISAKMP | AES, grupo DH2, pre-share |
| Tráfico cifrado (cripto-ACL) | 192.168.20.0/24 (VLAN20) ↔ 172.16.51.0/24 (segmento Host_1) |

## Control de acceso hacia Host_1 (ACL de filtrado)

La cripto-ACL define qué se cifra, pero no filtra tráfico. Para restringir el acceso al segmento de Host_1 únicamente a VLAN20 (conforme al requisito), se aplica una ACL extendida en CORE_R1, saliente hacia BR1:

| Parámetro | Valor |
|---|---|
| Nombre | FILTRO-HOST1 |
| Ubicación | CORE_R1, Ethernet0/0 (out), cercana al origen |
| Regla 1 | permit ip 192.168.20.0/24 → 172.16.51.0/24 (VLAN20 permitida) |
| Regla 2 | deny ip any → 172.16.51.0/24 (resto bloqueado: VLAN21, VLAN22) |
| Regla 3 | permit ip any any (resto del tráfico sin afectar) |
