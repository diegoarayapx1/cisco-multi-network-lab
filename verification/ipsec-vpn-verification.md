# Verificación VPN IPSec Site-to-Site

## Diseño

Túnel IPSec site-to-site entre BR1 (200.10.10.2) y BR2 (200.20.20.2), cifrando exclusivamente el tráfico entre VLAN20 (192.168.20.0/24, donde está PC_VLAN20) y el segmento de Host_1 (172.16.51.0/24, conectado a CORE_R2).

El control de acceso (qué tráfico puede *llegar* a Host_1) se implementa con una ACL de filtrado separada en CORE_R1 — ver sección "Control de acceso" más abajo y `lessons-learned.md`.

## Fase 1 — ISAKMP

```
BR1#sh crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
200.20.20.2     200.10.10.2     QM_IDLE           1001 ACTIVE
```

Fase 1 negociada correctamente (`QM_IDLE`, `ACTIVE`).

## Fase 2 — IPSec SA bidireccional confirmada

Ambos extremos cifran y descifran la misma cantidad de paquetes, con el par de SPI cruzado (el outbound de un lado es el inbound del otro):

```
BR1#show crypto ipsec sa
local  ident  (addr/mask/prot/port): (192.168.20.0/255.255.255.0/0/0)
remote ident  (addr/mask/prot/port): (172.16.51.0/255.255.255.0/0/0)
current_peer 200.20.20.2 port 500
    #pkts encaps: 9, #pkts encrypt: 9, #pkts digest: 9
    #pkts decaps: 9, #pkts decrypt: 9, #pkts verify: 9
    current outbound spi: 0xE5194808(3843639304)
    inbound esp sas:
     spi: 0xED79FF8A(3984195466)
      transform: esp-aes esp-sha-hmac
      in use settings ={Tunnel,}
```

```
BR2#show crypto ipsec sa | include local ident|remote ident|encaps|decaps|spi
remote ident (addr/mask/prot/port): (192.168.20.0/255.255.255.0/0/0)
    #pkts encaps: 9, #pkts encrypt: 9, #pkts digest: 9
    #pkts decaps: 9, #pkts decrypt: 9, #pkts verify: 9
    current outbound spi: 0xED79FF8A(3984195466)
     spi: 0xE5194808(3843639304)
```

`encaps: 9 / decaps: 9` en ambos extremos, con SPI espejo (`0xE5194808` ↔ `0xED79FF8A`): tráfico real cifrado en las dos direcciones.

## Prueba de conectividad — control positivo (VLAN20)

```
PC_VLAN20>ping 172.16.51.2
Sending 5, 100-byte ICMP Echos to 172.16.51.2, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5)

PC_VLAN20>ping 172.16.51.2
Sending 5, 100-byte ICMP Echos to 172.16.51.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 4/4/7 ms
```

El primer paquete del primer intento se pierde mientras se negocia la SA on-demand (comportamiento esperado); el segundo intento da 100%.

## Control de acceso — control negativo (VLAN21)

El requisito especifica que solo VLAN20 puede alcanzar a Host_1. Como una cripto-ACL define qué se cifra pero **no filtra** tráfico, se implementó una ACL extendida de filtrado en CORE_R1 (saliente hacia BR1) para descartar todo origen distinto de VLAN20 hacia 172.16.51.0/24.

```
CORE_R1 — ip access-list extended FILTRO-HOST1
 permit ip 192.168.20.0 0.0.0.255 172.16.51.0 0.0.0.255
 deny   ip any 172.16.51.0 0.0.0.255
 permit ip any any
 (aplicada: interface Ethernet0/0, ip access-group FILTRO-HOST1 out)
```

Resultado del ping desde VLAN21 hacia Host_1 (debe fallar):

```
PC_VLAN21#ping 172.16.51.2
Sending 5, 100-byte ICMP Echos to 172.16.51.2, timeout is 2 seconds:
U.U.U
Success rate is 0 percent (0/5)

PC_VLAN21#ping 172.16.51.2
Sending 5, 100-byte ICMP Echos to 172.16.51.2, timeout is 2 seconds:
U.U.U
Success rate is 0 percent (0/5)
```

`U.U.U` → 0% de forma consistente: el router descarta el tráfico de VLAN21 hacia Host_1 (devuelve unreachable), mientras VLAN20 sigue pasando al 100%. El control de acceso queda demostrado **por diseño**, no por ausencia de ruta.

## Configuración aplicada (resumen)

**BR1:**
```
crypto isakmp policy 10
 encr aes
 authentication pre-share
 group 2
crypto isakmp key CISCO address 200.20.20.2

crypto ipsec transform-set TSET esp-aes esp-sha-hmac
 mode tunnel

crypto map CMAP 10 ipsec-isakmp
 set peer 200.20.20.2
 set transform-set TSET
 match address ACL-VPN

ip access-list extended ACL-VPN
 permit ip 192.168.20.0 0.0.0.255 172.16.51.0 0.0.0.255

interface Ethernet0/0
 crypto map CMAP
```

**BR2 (configuración espejo):**
```
crypto isakmp key CISCO address 200.10.10.2

crypto map CMAP 10 ipsec-isakmp
 set peer 200.10.10.2
 set transform-set TSET
 match address ACL-VPN

ip access-list extended ACL-VPN
 permit ip 172.16.51.0 0.0.0.255 192.168.20.0 0.0.0.255

interface Ethernet0/0
 crypto map CMAP
```
