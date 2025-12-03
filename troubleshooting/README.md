# Troubleshooting Cisco — Guía magistral Cisco 1

Metodología estructurada de diagnóstico para redes Cisco. Incluye flujos paso a paso, comandos clave, interpretación de salidas y tablas de Síntomas/Causas/Acciones. Diseñado para capa de acceso/distribución y entornos de laboratorio/producción a nivel CCNA/Cisco 1.

Referencias recomendadas:
- Fundamentos IOS y operación segura: ver [ios/README.md](ios/README.md)
- Enrutamiento y servicios en routers: ver [routers/README.md](routers/README.md)
- Switching de acceso y distribución: ver [switches/README.md](switches/README.md)


## Índice
1. Principios y metodología de troubleshooting
2. Kit de herramientas en IOS (show, debug, logs, filtros)
3. Flujos de verificación rápidos (First 10 commands)
4. Capa 1: enlaces físicos, velocidad/duplex, errores
5. Capa 2: VLANs, Trunks, DTP, STP/RSTP, Port-Security, EtherChannel
6. Capa 3: direccionamiento, ARP/ND, routing estático y dinámico
7. Servicios de red: DHCP, NAT/PAT, DNS, NTP, SSH, Syslog
8. ACLs y seguridad de acceso (líneas, VTY, SSH)
9. Tablas de síntomas/causas/acciones (SCA) por dominio
10. Captura de paquetes (EPC) y técnicas de medición
11. Checklist previo/post cambio y buenas prácticas operativas


## 1. Principios y metodología de troubleshooting

Objetivo: resolver incidentes de forma predecible, rápida y documentable.

Marco general (ciclo):
- Definir el alcance del problema: quién, qué, cuándo, dónde, impacto.
- Recolectar hechos: topología, cambios recientes, logs, métricas.
- Formular hipótesis: ordenadas por probabilidad/impacto.
- Probar hipótesis: una a la vez, cambios mínimos y reversibles.
- Implementar la corrección: con ventana de cambio si aplica.
- Verificar y documentar: validar KPIs y actualizar la base de conocimiento.

Estrategias:
- De arriba hacia abajo (Aplicación → Capa 1) o por división binaria (divide y vencerás).
- Empezar por lo más probable y lo más simple (cables, shutdown, VLAN equivocada).
- Cambios seguros: use `terminal length 0` para exportar salidas y `do show` en configuración.

Reglas de oro:
- Nunca cambie más de una variable a la vez.
- Registre evidencia (comandos y salidas) antes/después.
- Si el entorno es productivo, active logs y NTP para correlación de eventos (ver [ios/README.md](ios/README.md)).


## 2. Kit de herramientas en IOS (show, debug, logs, filtros)

Comandos fundamentales de estado:
```bash
show clock detail
show running-config | section <objeto>
show ip interface brief
show ipv6 interface brief
show interfaces status
show interfaces counters errors
show interfaces <if> | include line|rate|error|CRC|drops
show controllers <if>    ! ver problemas físicos (dependiente de plataforma)
```

Operación de logs:
```bash
terminal length 0
terminal monitor              ! ver logs en sesiones VTY
show logging
logging buffered 16384 informational
! logging host 192.0.2.50     ! servidor syslog
service timestamps log datetime msec localtime show-timezone
```

Filtrado de salida:
- `| include <patrón>`
- `| exclude <patrón>`
- `| begin <patrón>`

Uso de `debug` (con cautela):
```bash
debug ip packet detail        ! CUIDADO en producción
debug ip icmp
debug arp
undebug all
```
- Regule el nivel de debug y prefiera ventanas de baja carga.
- Asegure `terminal monitor` y `logging buffered` antes de debug.


## 3. Flujos de verificación rápidos (First 10 commands)

Check de 60–120 segundos típico para “no hay conectividad”:
1) `show ip interface brief` — ¿interfaz up/up? ¿IP correcta?
2) `show interfaces <if>` — errores, CRC, input/output drops, duplex/velocidad.
3) `show vlan brief` (switch) — ¿puerto en VLAN correcta?
4) `show interfaces trunk` (si aplica) — ¿la VLAN viaja por el trunk?
5) `show mac address-table dynamic` — ¿aprendizaje de MAC coherente?
6) `show arp` / `show ip arp` — resolución IP→MAC presente/ausente.
7) `ping <puerta de enlace>` — prueba local; luego `ping <IP remota>`.
8) `traceroute <destino>` — verificar salto por salto (routing/NAT).
9) `show ip route` / `show ipv6 route` — ¿existe la ruta?
10) `show access-lists` / `show ip interface <if>` — ACL aplicada y sentido correctos.

Si hay cambios recientes: `show archive` y `show logging` para correlación.


## 4. Capa 1: enlaces físicos, velocidad/duplex, errores

Síntomas:
- Interfaz “down/down” o “administratively down”.
- Errores: `CRC`, `input errors`, `collisions`, `late collisions`.
- Flaps de enlace (sube/baja frecuentemente).

Acciones:
- Verificar estado y descripción:
```bash
show interfaces status
show interfaces <if>
```
- Revisar negociación/forzado de velocidad/duplex y coherencia extremo a extremo:
```bash
interface g0/1
 speed 1000
 duplex full
```
- Cables/SFP: reemplazar, limpiar fibra, revisar longitud y tipo.
- Deshabilitar/activar para reinicializar:
```bash
interface g0/1
 shutdown
 no shutdown
```
- Ver logs de flapping/errdisable:
```bash
show logging
show errdisable recovery
```

Buenas prácticas:
- Estándar con `description` útil en todos los puertos.
- Homologar velocidad/duplex; evitar “duplex mismatch” (muchas `late collisions`/`CRC`).


## 5. Capa 2: VLANs, Trunks, DTP, STP/RSTP, Port-Security, EtherChannel

VLANs:
```bash
show vlan brief
show interfaces switchport
```
- Confirmar que el puerto de acceso pertenece a la VLAN correcta.
- Gateway en SVI correcto (switch L3): `show ip interface brief | include Vlan`.

Trunking/DTP:
```bash
show interfaces trunk
show interfaces <if> trunk
show interfaces <if> switchport
```
- Verificar VLAN nativa y allowed list. Evitar `dynamic desirable` no controlado.
- Solución típica: trunk estático + `switchport nonegotiate` (ver [switches/README.md](switches/README.md)).

STP/RSTP:
```bash
show spanning-tree summary
show spanning-tree vlan <id>
show spanning-tree interface <if> detail
```
- Root esperado por VLAN; puertos bloqueo/forward coherentes.
- En acceso: `spanning-tree portfast` y `bpduguard` habilitados. Causas de `err-disable` por BPDU Guard.

Port-Security:
```bash
show port-security interface <if>
show port-security address
```
- Violaciones (protect/restrict/shutdown). Sticky MAC incoherentes tras mover equipos.

EtherChannel:
```bash
show etherchannel summary
show lacp neighbor
show pagp neighbor
show interfaces port-channel <n>
```
- Revisar modo (LACP/PAgP/On) y consistencia de parámetros entre miembros:
  trunk/access, VLANs permitidas, nativa, velocidad/duplex.


## 6. Capa 3: direccionamiento, ARP/ND, routing estático y dinámico

Direccionamiento y ARP/ND:
```bash
show ip interface brief
show ip arp
show ipv6 neighbors
```
- IP/máscara/gateway correctos. Conflictos IP. ARP incompleto → problema L2 o gateway inaccesible.

Routing estático:
```bash
show ip route static
show ipv6 route static
```
- Rutas por defecto presentes (`0.0.0.0/0`, `::/0`). Ver next-hop y salida.
- Rutas flotantes (AD mayor) como respaldo.

OSPF:
```bash
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
```
- Vecinos FULL. Coincidencia de área, timers, MTU, autenticación.
- `passive-interface` accidental. `ip ospf network` punto a punto en enlaces WAN.

OSPFv3:
```bash
show ipv6 ospf neighbor
show ipv6 route ospf
```

EIGRP:
```bash
show ip eigrp neighbors
show ip eigrp topology
show ip route eigrp
```
- AS/Named mode coherente, K-values iguales, redes anunciadas correctas.

RIP:
```bash
show ip protocols
show ip rip database
show ip route rip
```
- Versión 2, `no auto-summary`, interfaces no pasivas donde corresponde.


## 7. Servicios de red: DHCP, NAT/PAT, DNS, NTP, SSH, Syslog

DHCP (servidor):
```bash
show ip dhcp pool
show ip dhcp binding
debug ip dhcp server events
```
- Exclusiones, `default-router`, `dns-server`, ámbito correcto.
- Si el router no es servidor, `ip helper-address` en la interfaz de entrada (relay).

NAT/PAT:
```bash
show ip nat translations
show ip nat statistics
```
- ACL fuente y `ip nat inside/outside` correctos en interfaces.
- `ip nat inside source list ... interface <wan> overload` activo para PAT.

DNS:
- Ver si el host final resuelve; en router: `ip name-server <DNS>`, `ping nombre`.

NTP:
```bash
show ntp associations
show clock detail
```
- Fuentes NTP reachables; timezone/summer-time correctos.

SSH:
```bash
show ip ssh
show running-config | section line vty
```
- `ip domain-name`, RSA key, versión 2, `transport input ssh`, `login local`.

Syslog:
```bash
show logging
```
- `logging buffered`, `logging host`, `logging source-interface`. Nivel `trap` apropiado.


## 8. ACLs y seguridad de acceso (líneas, VTY, SSH)

Errores típicos:
- ACL aplicada en interfaz o dirección incorrecta (in/out).
- Falta de `permit` para tráfico de retorno o protocolos auxiliares.
- `implicit deny any` al final.

Diagnóstico:
```bash
show access-lists
show ip interface <if>
```

Acciones:
- Reordenar ACEs, probar con matches específicos (`log` para contadores).
- Aplicar `ip access-group <name> in/out` en la interfaz correcta.
- Probar conectividad tras ajustar.


## 9. Tablas de síntomas/causas/acciones (SCA) por dominio

Capa 1:
- Síntoma: CRC/late collisions altos
  - Causas: duplex mismatch, cable dañado, interferencia
  - Acciones: forzar duplex/velocidad coherentes, reemplazar cable/SFP, revisar ruta física

- Síntoma: Enlace flapping
  - Causas: transceiver defectuoso, potencia óptica baja, switch vecino reiniciando
  - Acciones: `show logging`, cambiar SFP, medir potencia, estabilizar negociación

Capa 2 (VLAN/Trunk/STP):
- Síntoma: Host en VLAN incorrecta (sin DHCP, sin puerta de enlace)
  - Causas: puerto mal asignado, SVI down, trunk sin VLAN permitida
  - Acciones: `show vlan brief`, `show interfaces trunk`, corregir access/trunk/SVI

- Síntoma: Loop de capa 2 (tempestad broadcast)
  - Causas: PortFast en uplink, ausencia de STP en segmento
  - Acciones: activar STP Rapid, BPDU Guard en acceso, Loop Guard en uplinks, storm-control

- Síntoma: Puerto err-disabled
  - Causas: bpduguard, port-security, link-flap, storm-control
  - Acciones: `show errdisable recovery`, corregir causa, habilitar recovery si procede

Port-Security:
- Síntoma: PC sin red tras moverlo de puerto
  - Causas: sticky MAC previa bloqueando
  - Acciones: `clear port-security sticky interface <if>`, ajustar `maximum`

EtherChannel:
- Síntoma: Solo parte del PoCH up
  - Causas: VLANs/duplex/velocidad inconsistentes, modos LACP/PAgP dispares
  - Acciones: uniformizar configuración, revisar `show etherchannel summary`

Capa 3 / Routing:
- Síntoma: Ping al gateway falla
  - Causas: VLAN/SVI down, ARP incompleto
  - Acciones: `show ip int brief`, `show arp`, corregir VLAN/SVI y conectividad L2

- Síntoma: Traceroute se detiene en borde
  - Causas: falta ruta por defecto/NAT, ACL en borde
  - Acciones: `show ip route`, `show ip nat statistics`, `show access-lists`

OSPF/EIGRP:
- Síntoma: Vecinos no alcanzan FULL
  - Causas: timers/MTU/área/K-values/AS desalineados, passive-interface
  - Acciones: comparar params por interfaz, ajustar `ip ospf network`, remover passive

DHCP:
- Síntoma: Clientes sin IP
  - Causas: exclusiones mal puestas, helper ausente, pool agotado
  - Acciones: `show ip dhcp binding/pool`, `ip helper-address`, ampliar pool

NAT/PAT:
- Síntoma: No hay traducciones
  - Causas: ACL fuente no coincide, inside/outside mal asignado
  - Acciones: corregir ACL, configurar `ip nat inside/outside` extremos correctos


## 10. Captura de paquetes (EPC) y técnicas de medición

Embedded Packet Capture (cuando el IOS lo soporte):
```bash
monitor capture buffer BUF size 2048 max-size 1518 circular
monitor capture point ip cef CAP g0/1 both
monitor capture point associate CAP BUF
monitor capture start
! Reproducir el problema...
monitor capture stop
monitor capture export tftp://192.168.10.20/epc-g0-1.pcap
monitor capture clear
```
- Use filtros por acceso/host/puerto si es viable (según versión IOS).
- Analice el PCAP en Wireshark para validar flags, TTL, DSCP, retransmisiones.

Mediciones:
- Latencia y pérdida: `ping repeat 100 size 1400`, `traceroute`.
- Throughput (en laboratorio): iPerf entre hosts, monitoreo en `show interfaces` (load/rate).
- Contadores de cola/descartes (si disponibles) para congestión.


## 11. Checklist previo/post cambio y buenas prácticas operativas

Antes del cambio:
- Confirmar ventana/impacto y plan de reversión.
- Registrar `show run`, `show ip int brief`, `show cdp neighbors`, `show vlan brief`, `show interfaces trunk`, `show ip route`.
- Activar `logging buffered` y verificar NTP/clock.

Durante:
- Cambiar una variable por vez.
- Documentar cada modificación y razón.

Después:
- Validar conectividad L2/L3: ping a gateway, a destinos clave, traceroute.
- Examinar tablas: MAC, ARP, routing, NAT, DHCP bindings.
- Revisar logs de eventos y counters de interfaz en carga.

Buenas prácticas:
- Estandarizar plantillas de acceso/trunk/port-security (ver [switches/README.md](switches/README.md)).
- Usar AAA/SSH/banners/timeouts consistentes (ver [ios/README.md](ios/README.md)).
- Versionado de configuraciones con `archive` y respaldos off-box.
- Mantener “known good state” (baseline) y diffs de cambio bien documentados.

Apéndice — Comandos “de bolsillo”:
```bash
show cdp neighbors detail
show lldp neighbors detail
show ip interface brief
show ipv6 interface brief
show interfaces status
show interfaces counters errors
show vlan brief
show interfaces trunk
show mac address-table dynamic
show ip arp
show ipv6 neighbors
show ip route
show ipv6 route
show ip protocols
show ip eigrp neighbors
show ip ospf neighbor
show ip nat translations
show ip dhcp binding
show logging
```