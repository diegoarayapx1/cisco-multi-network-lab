# Verificación BGP

## Estado de vecinos en el ISP (AS100)

```
ISP#sh ip bgp summary
BGP router identifier 100.100.100.100, local AS number 100
BGP table version is 13, main routing table version 13
12 network entries using 1680 bytes of memory
12 path entries using 960 bytes of memory

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
200.10.10.2     4          200      89      93       13    0    0 01:17:43        7
200.20.20.2     4          300      91      94       13    0    0 01:17:42        4
```

Ambas sesiones eBGP establecidas y estables (Up/Down con uptime largo, sin reset). El ISP recibe 7 prefijos de BR1 (AS200) y 4 de BR2 (AS300). `BGP table version 13` consistente en los tres routers, indicando convergencia total.

## Vecino BGP en BR1 (AS200)

```
BR1#sh ip bgp summary
BGP router identifier 1.1.1.1, local AS number 200
BGP table version is 13, main routing table version 13

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
200.10.10.1     4          100      94      90       13    0    0 01:18:15        5
```

## Vecino BGP en BR2 (AS300)

```
BR2#sh ip bgp summary
BGP router identifier 2.2.2.2, local AS number 300
BGP table version is 13, main routing table version 13

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
200.20.20.1     4          100      96      92       13    0    0 01:18:45        8
```

BR1 recibe 5 prefijos y BR2 recibe 8 desde el ISP, conforme a las redes visibles desde cada lado.

## Tabla BGP completa en el ISP (tras redistribución OSPF→BGP)

```
ISP#sh ip bgp
     Network          Next Hop            Metric LocPrf Weight Path
 *>  8.8.8.8/32       0.0.0.0                  0         32768 i
 *>  172.16.50.0/30   200.20.20.2              0             0 300 ?
 *>  172.16.51.0/30   200.20.20.2            100             0 300 ?
 *>  172.16.52.0/30   200.20.20.2            100             0 300 ?
 *>  192.168.20.0     200.10.10.2            100             0 200 ?
 *>  192.168.21.0     200.10.10.2            100             0 200 ?
 *>  192.168.22.0     200.10.10.2            100             0 200 ?
 *>  192.168.50.0/30  200.10.10.2              0             0 200 ?
 *>  192.168.51.0/30  200.10.10.2            100             0 200 ?
 *>  192.168.52.0/30  200.10.10.2            100             0 200 ?
```

Todas las redes internas de ambos AS son visibles desde el ISP gracias a la redistribución OSPF→BGP en BR1 y BR2.

## Configuración aplicada (resumen)

**ISP (AS100):**
```
router bgp 100
 bgp router-id 100.100.100.100
 network 8.8.8.8 mask 255.255.255.255
 neighbor 200.10.10.2 remote-as 200
 neighbor 200.20.20.2 remote-as 300
```

**BR1 (AS200):**
```
router bgp 200
 bgp router-id 1.1.1.1
 network 200.10.10.0 mask 255.255.255.252
 neighbor 200.10.10.1 remote-as 100
 redistribute ospf 1 metric 100
```

**BR2 (AS300):**
```
router bgp 300
 bgp router-id 2.2.2.2
 network 200.20.20.0 mask 255.255.255.252
 neighbor 200.20.20.1 remote-as 100
 redistribute ospf 1 metric 100
```
