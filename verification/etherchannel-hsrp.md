# Verificación EtherChannel y HSRP

## EtherChannel LACP

### Estado actual — post-corrección (Sesión 2)

```
Dist_SWA#sh etherchannel summary
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SU)         LACP      Et1/0(P)    Et1/1(P)
2      Po2(SU)         LACP      Et0/1(P)    Et0/2(P)
```

```
Dist_SWB#sh etherchannel summary
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SU)         LACP      Et1/0(P)    Et1/1(P)
3      Po3(SU)         LACP      Et0/1(P)    Et0/2(P)
```

```
Acceso_SW1#sh etherchannel summary
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
2      Po2(SU)         LACP      Et0/0(P)    Et0/1(P)
3      Po3(SU)         LACP      Et0/2(P)    Et0/3(P)
```

Los tres Port-Channels (Po1, Po2, Po3) quedan con **ambos** miembros en estado `(P)` (bundled), en los tres switches.

> **Corrección respecto a la verificación anterior:** una versión previa de este documento reportaba un puerto `(P)` y otro `(s)` (suspended) por canal en los tres switches, atribuido a "un solo enlace físico real disponible por canal en el entorno IOU Web". Ese diagnóstico era incorrecto. El netmap real de IOU sí provee dos enlaces físicos por cada Port-Channel; el puerto en `(s)` era el resultado de interfaces físicas asignadas al `channel-group` equivocado (mapeo cruzado entre Po1, Po2 y Po3). Detalle completo del mapeo y la corrección en `lessons-learned.md`, Sesión 2, hallazgo 1.

---

## HSRP IPv4 e IPv6

### Dist_SWA (Active)

```
Dist_SWA#show standby brief
Interface   Grp  Pri P State   Active   Standby         Virtual IP
Vl20        20   110 P Active  local    192.168.20.3    192.168.20.1
Vl20        120  110 P Active  local    FE80::A8BB:CCFF:FE80:700   FE80::5:73FF:FEA0:78
Vl21        21   110 P Active  local    192.168.21.3    192.168.21.1
Vl21        121  110 P Active  local    FE80::A8BB:CCFF:FE80:700   FE80::5:73FF:FEA0:79
Vl22        22   110 P Active  local    192.168.22.3    192.168.22.1
Vl22        122  110 P Active  local    FE80::A8BB:CCFF:FE80:700   FE80::5:73FF:FEA0:7A
```

### Dist_SWB (Standby)

```
Dist_SWB#show standby brief
Interface   Grp  Pri P State    Active           Standby   Virtual IP
Vl20        20   100 P Standby  192.168.20.2     local     192.168.20.1
Vl20        120  100 P Standby  FE80::A8BB:CCFF:FE80:600   local   FE80::5:73FF:FEA0:78
Vl21        21   100 P Standby  192.168.21.2     local     192.168.21.1
Vl21        121  100 P Standby  FE80::A8BB:CCFF:FE80:600   local   FE80::5:73FF:FEA0:79
Vl22        22   100 P Standby  192.168.22.2     local     192.168.22.1
Vl22        122  100 P Standby  FE80::A8BB:CCFF:FE80:600   local   FE80::5:73FF:FEA0:7A
```

Convergencia confirmada en los seis grupos (20/21/22 IPv4 y 120/121/122 IPv6): Dist_SWA Active (prioridad 110) en todos, Dist_SWB Standby (prioridad 100) en todos, sin estados "unknown".

### Log de convergencia en tiempo real

```
*Jun 30 07:12:40.755: %HSRP-5-STATECHANGE: Vlan22 Grp 122 state Speak -> Standby
```

---

## Configuración aplicada (resumen, ejemplo VLAN20 en Dist_SWA)

```
interface Vlan20
 standby version 2
 standby 20 ip 192.168.20.1
 standby 20 priority 110
 standby 20 preempt
 standby 120 ipv6 autoconfig
 standby 120 priority 110
 standby 120 preempt
```

## Mapeo EtherChannel verificado contra netmap de IOU

```
Po1 (Dist_SWA ↔ Dist_SWB):   Et1/0 + Et1/1 en ambos extremos
Po2 (Dist_SWA ↔ Acceso_SW1): Et0/1 + Et0/2 (SWA)  ↔  Et0/0 + Et0/1 (Acceso_SW1)
Po3 (Dist_SWB ↔ Acceso_SW1): Et0/1 + Et0/2 (SWB)  ↔  Et0/2 + Et0/3 (Acceso_SW1)
```
