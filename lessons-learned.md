# Lecciones aprendidas y troubleshooting

Este documento registra los problemas reales encontrados durante la implementación, su diagnóstico y su solución. El objetivo es mostrar el proceso de troubleshooting, no solo el resultado final.

---

## Sesión 1 — Implementación inicial (BGP, OSPF, VPN IPSec, endpoints, HSRP, EtherChannel)

### 1. ACL de la VPN IPSec apuntando a la subred incorrecta

**Síntoma:** El túnel ISAKMP se establecía correctamente (`QM_IDLE`, estado `ACTIVE`), pero el ping desde PC_VLAN20 hacia Host_1 (172.16.51.2) fallaba al 100%.

**Diagnóstico:**
```
BR1# sh crypto ipsec sa
local  ident (addr/mask/prot/port): (192.168.20.0/255.255.255.0/0/0)
remote ident (addr/mask/prot/port): (172.16.50.0/255.255.255.0/0/0)
```

La ACL `ACL-VPN` en BR1 tenía configurado el segmento `172.16.50.0/24` (el enlace BR2↔CORE_R2) en lugar de `172.16.51.0/24` (el segmento real donde está Host_1, conectado a CORE_R2). El tráfico interesante nunca matcheaba.

**Solución:**
```
BR1(config)# ip access-list extended ACL-VPN
BR1(config-ext-nacl)# no permit ip 192.168.20.0 0.0.0.255 172.16.50.0 0.0.0.255
BR1(config-ext-nacl)# permit ip 192.168.20.0 0.0.0.255 172.16.51.0 0.0.0.255
```

Se corrigió la ACL espejo en BR2 de la misma forma.

> **Nota (Sesión 2):** este mismo error reapareció idéntico tras un apagado completo de la VM de IOU Web. Ver Sesión 2, hallazgo 4 — la causa no fue repetir el error, sino que el Initial Config guardado en IOU Web no reflejaba esta corrección.

---

### 2. BGP no propagaba las redes internas (LAN) a través del ISP

**Síntoma:** BGP establecía sesión correctamente entre los tres AS, pero solo se anunciaban las redes de los enlaces punto a punto (`200.10.10.0/30`, `200.20.20.0/30`). Las redes internas (VLANs, segmentos de Host) no eran visibles desde el otro extremo.

**Diagnóstico:** Por defecto, BGP solo anuncia lo que se declara explícitamente con `network` o lo que se redistribuye desde otro protocolo. BR1 y BR2 no tenían redistribución desde OSPF.

**Solución:**
```
BR1(config)# router bgp 200
BR1(config-router)# redistribute ospf 1 metric 100

BR2(config)# router bgp 300
BR2(config-router)# redistribute ospf 1 metric 100
```

---

### 3. Switches de distribución (Dist_SWA/SWB) sin ruta hacia el exterior

**Síntoma:** Tras corregir BGP, el ping desde PC_VLAN21 hacia el lado derecho de la red seguía fallando, incluso antes de llegar a BR1.

**Diagnóstico:**
```
Dist_SWA# sh ip route
```
Dist_SWA solo tenía rutas OSPF internas (VLANs y enlaces de área 1). No conocía ninguna red externa (172.16.x.x, 200.x.x.x) porque BR1 no estaba redistribuyendo BGP hacia OSPF — solo se había hecho el sentido OSPF→BGP, no el inverso.

**Solución:**
```
BR1(config)# router ospf 1
BR1(config-router)# redistribute bgp 200 subnets

BR2(config)# router ospf 1
BR2(config-router)# redistribute bgp 300 subnets
```

Esta fue la lección más importante del lab: **la redistribución mutua debe hacerse en ambos sentidos** (OSPF→BGP y BGP→OSPF) en cada punto de borde, o el dominio interno queda aislado del mundo exterior aunque BGP funcione perfectamente "hacia afuera".

> **Nota (Sesión 2):** la pérdida de esta redistribución en BR1 (sentido OSPF→BGP y BGP→OSPF) fue exactamente el hallazgo 3 de la Sesión 2, recurrente por la misma causa raíz documentada ahí.

---

### 4. Redistribución BGP→OSPF aplicada pero sin efecto inmediato

**Síntoma:** Tras aplicar `redistribute bgp 300 subnets` en BR2, `sh ip ospf database external` seguía vacío y CORE_R2 no recibía las rutas.

**Diagnóstico:** El proceso OSPF no regeneró los LSA de tipo 5 inmediatamente con la configuración por defecto.

**Solución:**
```
BR2(config-router)# no redistribute bgp 300 subnets
BR2(config-router)# redistribute bgp 300 subnets metric 1 metric-type 2
BR2# clear ip ospf process
```
Forzar un `clear ip ospf process` regeneró la base de datos y los LSA tipo 5 se propagaron correctamente.

---

### 5. Switches IOS usados como "PC" no enrutaban el tráfico de retorno

**Síntoma:** Los endpoints (PC_VLAN20, PC_VLAN21, Host_1, Host_2) son switches IOS de capa 2 sin soporte de SVI (`interface vlan 1` no es un comando válido en ese modelo). Tras configurar la IP en la interfaz física y el `ip default-gateway`, el ping hacia el gateway local funcionaba, pero el ping hacia redes remotas fallaba al 100% aunque la ruta de ida llegaba correctamente al destino.

**Diagnóstico:** `ip default-gateway` solo es efectivo cuando el dispositivo **no** tiene `ip routing` habilitado (modo host). Sin `ip routing`, el switch no puede procesar ni reenviar paquetes IP que no sean para sí mismo, lo que impide el comportamiento de gateway por defecto esperado para tráfico saliente más allá de la subred local en este modelo de IOS.

**Solución:**
```
ip routing
ip route 0.0.0.0 0.0.0.0 <gateway>
```
Se reemplazó `ip default-gateway` por `ip routing` + una ruta estática por defecto explícita en los cuatro endpoints.

---

### 6. HSRP IPv4 e IPv6 no pueden compartir el mismo número de grupo (IOS 15.2)

**Síntoma:** Al intentar agregar `standby 20 ipv6 autoconfig` sobre la interfaz Vlan20, que ya tenía configurado `standby 20` para IPv4, el comando no se aplicaba como se esperaba.

**Diagnóstico:** Limitación conocida de esta versión de IOS en switches L3: un mismo grupo HSRP no soporta simultáneamente un VIP IPv4 y configuración IPv6 autoconfig de forma estable.

**Solución:** Se usaron grupos HSRP separados por familia de protocolo en la misma interfaz:
- Grupo 20/21/22 → IPv4
- Grupo 120/121/122 → IPv6 (autoconfig)

Misma prioridad y preempt en ambos grupos por VLAN, manteniendo Dist_SWA como Active (110) y Dist_SWB como Standby (100) de forma consistente en ambas familias.

---

### 7. EtherChannel LACP con un puerto suspendido por canal

**Síntoma:** `sh etherchannel summary` mostraba consistentemente un puerto en estado `P` (bundled) y otro en estado `s` (suspended) en cada Port-Channel (Po1, Po2, Po3), en los tres switches involucrados.

**Diagnóstico original (incorrecto — ver corrección en Sesión 2):** Se interpretó como una limitación del entorno IOU Web (solo un enlace físico real por Port-Channel), no como error de configuración, ya que la sintaxis `channel-group X mode active` parecía simétrica.

> **⚠️ Corrección (Sesión 2):** este diagnóstico fue erróneo. El netmap real de IOU sí provee **dos enlaces físicos por cada Port-Channel** (confirmado: `Po1` = e1/0+e1/1, `Po2`/`Po3` = e0/1+e0/2 en cada extremo). El puerto en `(s)` no era una limitación del entorno, sino el resultado de interfaces físicas asignadas al `channel-group` equivocado — por ejemplo, una interfaz de Po1 quedaba metida en el `channel-group` de Po2, generando una agregación incompleta y un puerto sin contraparte coherente del otro lado. Al corregir la asignación interfaz por interfaz contra el netmap, **ambos** miembros de cada Port-Channel pasaron a `(P)` simultáneamente en los tres switches. Ver Sesión 2, hallazgo 1, para el detalle completo del mapeo y la corrección.

---

## Sesión 2 — Recuperación tras apagado completo de la VM de IOU Web

Tras cerrar el lab apagando la máquina virtual de IOU Web por completo (no solo cerrando el navegador), el lab —que ya estaba 100% verificado en la Sesión 1— dejó de funcionar. El diagnóstico reveló tres errores de configuración recurrentes (idénticos a los de la Sesión 1) más una causa raíz sistémica nueva.

### 1. Channel-groups cruzados en Dist_SWA / Dist_SWB → split-brain de HSRP

**Síntoma:** HSRP en estado **Active/Active** con **Standby: unknown** en ambos switches de distribución. El enlace L2 directo Dist_SWA↔Dist_SWB (Po1, "HSRP PO1" en el diagrama) no transportaba tráfico HSRP.

**Diagnóstico:** Se mapeó el netmap real de IOU línea por línea contra el cableado lógico esperado:

```
6:1/0  7:1/0   → Dist_SWA e1/0 ↔ Dist_SWB e1/0   (Po1)
6:1/1  7:1/1   → Dist_SWA e1/1 ↔ Dist_SWB e1/1   (Po1)
6:0/1  10:0/0  → Dist_SWA e0/1 ↔ Acceso_SW1 e0/0 (Po2)
6:0/2  10:0/1  → Dist_SWA e0/2 ↔ Acceso_SW1 e0/1 (Po2)
7:0/1  10:0/2  → Dist_SWB e0/1 ↔ Acceso_SW1 e0/2 (Po3)
7:0/2  10:0/3  → Dist_SWB e0/2 ↔ Acceso_SW1 e0/3 (Po3)
```

`show etherchannel summary` confirmó que ese mapeo no se respetaba: en Dist_SWA, e0/1 estaba en `channel-group 1` (debía ser 2), e1/1 en `channel-group 2` (debía ser 1), y e1/0 sin channel-group asignado. En Dist_SWB, e0/1 en `channel-group 2` (debía ser 3), e1/0 en `channel-group 2` (debía ser 1), e1/1 en `channel-group 3` (debía ser 1), y `interface Port-channel1` no existía en el running.

**Solución:** Reasignación de cada interfaz física a su channel-group correcto según el netmap, y creación de la interfaz lógica `Port-channel1` faltante en Dist_SWB.

**Verificación:**
```
Dist_SWA:  Po1(SU) Et1/0(P) Et1/1(P)  |  Po2(SU) Et0/1(P) Et0/2(P)
Dist_SWB:  Po1(SU) Et1/0(P) Et1/1(P)  |  Po3(SU) Et0/1(P) Et0/2(P)
Acceso_SW1: Po2(SU) Et0/0(P) Et0/1(P) |  Po3(SU) Et0/2(P) Et0/3(P)

show standby brief:
Dist_SWA: Active  | Standby = 192.168.20.3 / .21.3 / .22.3  (prio 110)
Dist_SWB: Standby | Active  = 192.168.20.2 / .21.2 / .22.2  (prio 100)
```

---

### 2. ACL cripto no-espejo en BR1 (recurrencia del hallazgo 1 de la Sesión 1)

Mismo síntoma y causa que la Sesión 1: ACL en `172.16.50.0` en vez de `172.16.51.0`. Mismo fix aplicado. Ver detalle en Sesión 1, hallazgo 1.

---

### 3. Redistribución mutua OSPF↔BGP faltante en BR1 (recurrencia de los hallazgos 2 y 3 de la Sesión 1)

Mismo síntoma y causa que la Sesión 1: BR1 sin `redistribute bgp 200 subnets` en OSPF ni `redistribute ospf 1 metric 100` en BGP, lo que dejaba a CORE_R1 y CORE_R2 sin ruta hacia la red del extremo opuesto (`% Network not in table` en ambos). Mismo fix aplicado. Ver detalle en Sesión 1, hallazgos 2 y 3.

---

### 4. Causa raíz sistémica — el Initial Config de IOU Web pisa la NVRAM al reiniciar la VM completa

**El patrón observado:** Los tres hallazgos anteriores de esta sesión ya habían sido corregidos y guardados con `write memory` en la Sesión 1. Tras apagar la VM completa de IOU Web y reabrir el lab, reaparecieron de forma idéntica — no parcialmente distintos, sino exactamente los mismos channel-groups cruzados, la misma ACL con `172.16.50.0`, y la misma falta de redistribución en BR1. Esto se repitió dos veces en la misma sesión de troubleshooting.

**Diagnóstico:** IOU Web arma cada lab a partir de un netmap/topología y un **Initial Config por dispositivo** (Manage → [dispositivo] → cuadro de texto Initial Config), independiente de la NVRAM del nodo en ejecución. `write memory` guarda en la NVRAM de la instancia activa, pero cuando la VM completa se apaga y se vuelve a encender, cada nodo arranca **releyendo el Initial Config**, no la NVRAM de la sesión anterior — sin importar cuántas veces se haya hecho `wr`. En este lab, los archivos de Initial Config habían quedado guardados con una versión anterior a las correcciones de la Sesión 1, así que cada reinicio completo de la VM revertía el lab a ese estado.

**Solución permanente:** Actualizar manualmente el campo Initial Config de cada dispositivo modificado (Dist_SWA, Dist_SWB, BR1) en la GUI de IOU Web, reemplazando su contenido por la configuración corregida y verificada — `write memory` por sí solo no sobrevive un apagado completo de la VM.

**Lección:** En IOU Web, persistencia de NVRAM (`wr`) y persistencia de Initial Config son dos mecanismos distintos. Para que un lab sobreviva un apagado completo de la máquina (no solo el cierre del navegador), el Initial Config de cada nodo modificado debe actualizarse manualmente en la GUI tras cada sesión de cambios — de lo contrario, el lab "se rompe solo" de forma reproducible, dando la falsa impresión de corrupción de NVRAM.

---

### 5. La cripto-ACL no controla acceso — VLAN21 llegaba a Host_1 sin restricción

**Síntoma:** El requisito especifica que solo VLAN20 debe alcanzar a Host_1. Al probar el control negativo tras completar el routing, el ping desde PC_VLAN21 hacia Host_1 dio **100% de éxito** — exactamente lo contrario de lo esperado.

**Diagnóstico:** Confusión conceptual entre dos mecanismos distintos. La cripto-ACL (`ACL-VPN`) **define qué tráfico se cifra**, no qué tráfico se permite. Un paquete que no matchea la cripto-ACL no se descarta: se enruta en claro. Al completar la redistribución BGP↔OSPF (hallazgo 3), VLAN21 obtuvo ruta hacia 172.16.51.0 y, como su tráfico simplemente no entraba al túnel pero sí tenía camino, llegaba sin cifrar. El 0% que mostraba la documentación anterior era un falso negativo: VLAN21 fallaba por **falta de ruta**, no por un control de acceso real.

**Solución:** Se implementó una ACL extendida de filtrado, separada de la cripto-ACL, aplicada en CORE_R1 saliente hacia BR1 (cercana al origen):

```
ip access-list extended FILTRO-HOST1
 permit ip 192.168.20.0 0.0.0.255 172.16.51.0 0.0.0.255
 deny   ip any 172.16.51.0 0.0.0.255
 permit ip any any
!
interface Ethernet0/0
 ip access-group FILTRO-HOST1 out
```

**Verificación:** PC_VLAN21 → Host_1 pasó a `U.U.U` (0%, descartado por la ACL), mientras PC_VLAN20 → Host_1 se mantuvo en 100%.

**Lección:** Cripto-ACL y ACL de filtrado son mecanismos independientes con propósitos distintos: una decide qué se *cifra* (`match address` en el crypto map), la otra decide qué se *permite o descarta* (`ip access-group` en una interfaz). Un `permit` en una cripto-ACL no abre acceso y un `deny` no lo bloquea. Restringir el acceso a un segmento requiere una ACL de filtrado explícita — y conviene validar el control negativo con routing completo, porque un "0%" por falta de ruta puede enmascarar la ausencia total de control de acceso.

---

### Resultado final verificado (Sesión 2)

```
PC_VLAN20>ping 172.16.51.2
Success rate is 100 percent (5/5), round-trip min/avg/max = 4/4/7 ms

BR1#show crypto ipsec sa
#pkts encaps: 9, #pkts decaps: 9
current outbound spi: 0xE5194808(3843639304)
inbound esp sas: spi 0xED79FF8A(3984195466), transform esp-aes esp-sha-hmac
in use settings ={Tunnel,}
```

EtherChannel con los tres switches en `(P)`, HSRP convergido (Dist_SWA Active prio 110 / Dist_SWB Standby prio 100), rutas O E2 en ambos cores, y túnel IPSec cifrando/descifrando tráfico real entre VLAN20 y el segmento de Host_1.

---

## Resumen de aprendizajes clave

1. La redistribución entre protocolos de enrutamiento debe pensarse en ambos sentidos (BGP↔OSPF), no solo en el sentido "hacia afuera".
2. Un túnel ISAKMP en estado ACTIVE no garantiza que el tráfico fluya: hay que verificar también que la ACL de tráfico interesante matchee exactamente la subred real de origen/destino con `sh crypto ipsec sa`.
3. Distinguir entre fallo de configuración real y limitación del entorno de laboratorio es una habilidad de troubleshooting tan importante como saber configurar el protocolo en sí — y a veces el primer diagnóstico está equivocado (ver corrección al hallazgo 7 de la Sesión 1).
4. `ip default-gateway` e `ip routing` son mutuamente excluyentes en su efecto: hay que elegir uno según si el dispositivo actúa como host o como router.
5. Antes de tocar un solo `channel-group` en vivo, hay que verificar el netmap físico real contra el cableado lógico esperado — reasignar grupos sobre la marcha sin ese mapeo previo genera cambios contradictorios.
6. En IOU Web, `write memory` (persistencia en NVRAM) y la persistencia del Initial Config son mecanismos independientes. Un lab que "se rompe solo" tras reabrir la VM completa probablemente no sea corrupción de NVRAM, sino un Initial Config desactualizado — hay que actualizarlo manualmente en la GUI tras cada sesión de cambios relevantes.
7. Una cripto-ACL define qué tráfico se cifra, no qué tráfico se permite. Restringir el acceso a un segmento requiere una ACL de filtrado separada (`ip access-group`). Al validar un control negativo, conviene hacerlo con el routing ya completo: un "0%" causado por falta de ruta puede ocultar que no existe ningún control de acceso real.
