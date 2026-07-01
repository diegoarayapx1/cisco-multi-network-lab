# Verificación OSPF

## Adyacencias en CORE_R1 (ABR área 0 / área 1)

```
CORE_R1#sh ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           1   FULL/BDR        00:00:34    192.168.50.1    Ethernet0/0
6.6.6.6           1   FULL/DR         00:00:37    192.168.52.2    Ethernet0/3
5.5.5.5           1   FULL/DR         00:00:32    192.168.51.2    Ethernet0/2
```

- `1.1.1.1` = BR1 (área 0)
- `5.5.5.5` = Dist_SWA (área 1)
- `6.6.6.6` = Dist_SWB (área 1)

Tres adyacencias en estado `FULL`, confirmando que CORE_R1 actúa como ABR entre el área 0 (hacia BR1) y el área 1 (hacia las dos switches de distribución).

## Adyacencia BR2 ↔ CORE_R2 (área 0)

```
BR2#sh ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
4.4.4.4           1   FULL/DR         00:00:39    172.16.50.2     Ethernet0/2
```

## Rutas inter-área en CORE_R1 (área 1 → propagadas)

```
CORE_R1#sh ip route ospf
O     192.168.20.0/24 [110/11] via 192.168.52.2, Ethernet0/3
                      [110/11] via 192.168.51.2, Ethernet0/2
O     192.168.21.0/24 [110/11] via 192.168.52.2, Ethernet0/3
                      [110/11] via 192.168.51.2, Ethernet0/2
O     192.168.22.0/24 [110/11] via 192.168.52.2, Ethernet0/3
                      [110/11] via 192.168.51.2, Ethernet0/2
```

Balanceo de carga ECMP entre Dist_SWA y Dist_SWB hacia las tres VLANs.

## Redistribución BGP→OSPF (rutas externas tipo E2)

Tras configurar `redistribute bgp 200 subnets` en BR1, las redes del lado AS300 se propagan como rutas externas E2 a los switches de distribución:

```
Dist_SWA#sh ip route ospf
O E2     172.16.50.0 [110/1] via 192.168.51.1, Ethernet0/0
O E2     172.16.51.0 [110/1] via 192.168.51.1, Ethernet0/0
O E2     172.16.52.0 [110/1] via 192.168.51.1, Ethernet0/0
O E2     200.20.20.0 [110/1] via 192.168.51.1, Ethernet0/0
```

Y en el sentido inverso, CORE_R2 aprende las VLANs del lado izquierdo:

```
CORE_R1#show ip route 172.16.51.0
Routing entry for 172.16.51.0/30
  Known via "ospf 1", distance 110, metric 1
  Tag 100, type extern 2, forward metric 10
  * 192.168.50.1, from 1.1.1.1, via Ethernet0/0

CORE_R2#show ip route 192.168.20.0
Routing entry for 192.168.20.0/24
  Known via "ospf 1", distance 110, metric 1
  Tag 100, type extern 2, forward metric 10
  * 172.16.50.1, from 2.2.2.2, via Ethernet0/1
```

Rutas de ida (172.16.51.0 en el dominio izquierdo) y de vuelta (192.168.20.0 en el dominio derecho) confirmadas como `O E2`, habilitando la conectividad bidireccional que el túnel IPSec necesita.

## Configuración aplicada (resumen)

**CORE_R1 (ABR):**
```
router ospf 1
 router-id 3.3.3.3
 network 192.168.50.0 0.0.0.3 area 0
 network 192.168.51.0 0.0.0.3 area 1
 network 192.168.52.0 0.0.0.3 area 1
```
Nota: se removió `passive-interface` en E0/2 y E0/3 (enlaces hacia Dist_SWA/SWB), ya que requieren formar adyacencia OSPF, no solo anunciar la red.

**BR1 / BR2 (redistribución bidireccional):**
```
! BR1
router ospf 1
 redistribute bgp 200 subnets
router bgp 200
 redistribute ospf 1 metric 100

! BR2
router ospf 1
 redistribute bgp 300 subnets metric 1 metric-type 2
router bgp 300
 redistribute ospf 1 metric 100
```
