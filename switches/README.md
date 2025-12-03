# Switches Cisco — Guía magistral Cisco 1

Guía completa para la configuración de switches Cisco a nivel CCNA/Cisco 1: VLANs, VTP, Trunking 802.1Q, DTP, STP/RSTP, EtherChannel (LACP/PAgP), Port-Security y Voice VLAN. Incluye comandos, explicación, verificación y mejores prácticas operativas.

Referencias previas recomendadas:
- Fundamentos de IOS: ver [ios/README.md](ios/README.md)
- Conceptos de enrutamiento y servicios: ver [routers/README.md](routers/README.md)


## Índice
1. Plantilla base de endurecimiento (hardening) para switches
2. VLANs: conceptos, creación, puertos access, SVI y verificación
3. VTP: dominios, modos, versión, password, pruning y buenas prácticas
4. Trunking 802.1Q: allowed VLANs, nativa, verificación y seguridad
5. DTP: modos, negociación y recomendaciones
6. STP/RSTP/MST: elección de root, PortFast, BPDU Guard/Filter, Root/Loop Guard
7. EtherChannel: PAgP, LACP, estático; consistencia y verificación
8. Port-Security: sticky MAC, violaciones, aging e interacción con Voice VLAN
9. Voice VLAN y QoS básica en acceso
10. Operación y diagnóstico: show/clear, errdisable, storm-control, CDP/LLDP
11. Plantillas rápidas por función (Access/Distribution)
12. Checklist final de verificación


## 1. Plantilla base de endurecimiento (hardening) para switches

Objetivo: acceso seguro, trazabilidad y comportamiento consistente.

```bash
hostname SW-AC-1
no ip domain-lookup
service timestamps debug datetime msec localtime show-timezone
service timestamps log datetime msec localtime show-timezone
service password-encryption
ip domain-name ejemplo.local

enable secret SecretoFuerte123
username admin privilege 15 secret SecretoFuerte123

line console 0
 logging synchronous
 exec-timeout 10 0
 login local
exit

line vty 0 4
 transport input ssh
 logging synchronous
 exec-timeout 10 0
 login local
exit

ip ssh version 2
crypto key generate rsa modulus 2048

! Registrar logs localmente y opcionalmente hacia un servidor
logging buffered 16384 informational
! logging host 192.0.2.50
! logging source-interface Vlan10

! Sincronización de hora para correlación de eventos
clock timezone CST -6 0
clock summer-time CDT recurring
! ntp server 192.0.2.10 prefer
```

Buenas prácticas:
- Limitar acceso remoto a SSH y considerar ACL con `access-class`.
- Activar `service password-encryption` (ofuscación básica) y preferir `secret`.
- Sincronizar hora (NTP) para logs coherentes. Ver [ios/README.md](ios/README.md).


## 2. VLANs: conceptos, creación, puertos access, SVI y verificación

Conceptos clave:
- VLAN (Virtual LAN) segmenta dominios de broadcast en L2.
- Tramas se etiquetan con 802.1Q en trunks; en puertos access no hay etiqueta.
- SVI (Switched Virtual Interface) provee gateway L3 en switches multilayer.

Crear VLANs y nombrarlas:
```bash
conf t
vlan 10
 name USUARIOS
vlan 20
 name VOZ
vlan 30
 name SERVIDORES
end
```

Asignar puertos access:
```bash
conf t
interface range fa0/1 - 24
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
 description ACCESO-USUARIOS
end
```

SVI (en switch capa 3) y gateway para gestión:
```bash
conf t
interface vlan 10
 ip address 192.168.10.2 255.255.255.0
 no shutdown
!
ip default-gateway 192.168.10.1   ! en switches L2; en multilayer usar routing
end
```

Verificación de VLANs y puertos:
```bash
show vlan brief
show interfaces status
show interfaces switchport
```

Notas operativas:
- Use `description` consistente para identificar función del puerto.
- Habilite `portfast` solo en acceso a hosts (nunca en enlaces a switches).


## 3. VTP: dominios, modos, versión, password, pruning y buenas prácticas

VTP (VLAN Trunking Protocol) propaga la base de datos de VLANs dentro de un dominio. Modos:
- Server: crea/borra VLANs y las anuncia.
- Client: aprende del server; no guarda en NVRAM.
- Transparent: no participa en base de datos, pero reenvía mensajes VTP; VLANs locales independientes.

Configuración típica:
```bash
conf t
vtp domain CAMPUS
vtp mode transparent
vtp version 2
! vtp password ClaveVTP
end
show vtp status
```

Pruning:
```bash
conf t
vtp pruning
end
```

Buenas prácticas:
- Por defecto, utilice `vtp mode transparent` para evitar impactos por número de revisión.
- Si usa VTP, controle el `revision number` y el `password` y documente topología.
- Active pruning para evitar tráfico innecesario en trunks (cuando aplica).


## 4. Trunking 802.1Q: allowed VLANs, VLAN nativa, verificación y seguridad

Crear un trunk estático:
```bash
conf t
interface g0/1
 switchport trunk encapsulation dot1q   ! si el hardware lo requiere
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 switchport trunk native vlan 999
 description TRUNK-A-Distribucion
 spanning-tree guard loop
end
```

Buenas prácticas:
- Definir VLAN nativa dedicada (ej. 999) y no usarla para usuarios.
- Restringir VLANs permitidas en cada trunk con `allowed vlan`.
- Monitorear mismatches de VLAN nativa y allowed lists.

Verificación:
```bash
show interfaces trunk
show interfaces g0/1 trunk
show spanning-tree vlan 10
```

Seguridad en trunk:
- Evitar tener puertos en `dynamic desirable` en redes mixtas.
- Usar `switchport nonegotiate` para desactivar DTP cuando corresponda (ver sección DTP).


## 5. DTP: modos, negociación y recomendaciones

DTP (Dynamic Trunking Protocol) negocia trunks entre switches Cisco.

Modos principales por puerto:
- access: fuerza puerto de acceso.
- trunk: fuerza trunk.
- dynamic desirable: intenta formar trunk si el otro extremo lo permite.
- dynamic auto: espera; formará trunk si el otro es desirable.
- nonegotiate: deshabilita DTP (se requiere trunk estático en el otro extremo).

Configuraciones típicas:
```bash
! Trunk estático y sin DTP (recomendado en enlaces punto a punto controlados)
interface g0/1
 switchport mode trunk
 switchport nonegotiate
 switchport trunk allowed vlan 10,20,30

! Puerto de acceso fijo (no negociable)
interface fa0/10
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
```

Buenas prácticas:
- Preferir trunk estático + `nonegotiate` en enlaces planificados.
- En acceso a hosts, forzar `mode access` para evitar formaciones indeseadas de trunk.


## 6. STP/RSTP/MST: elección de root, PortFast, BPDU Guard/Filter, Root/Loop Guard

Conceptos clave:
- STP evita bucles en capa 2 bloqueando enlaces redundantes.
- PVST+ (per-VLAN STP) y Rapid-PVST+ (RSTP por VLAN) son comunes en campus Cisco.
- Root Bridge: switch con menor prioridad+MAC por VLAN.

Establecer prioridades de root:
```bash
conf t
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20,30 priority 4096
! o usar macro:
! spanning-tree vlan 10 root primary
! spanning-tree vlan 20 root secondary
end
```

PortFast y BPDU Guard (en acceso a usuarios):
```bash
conf t
spanning-tree portfast default
spanning-tree bpduguard default
end
```

Root Guard (evita que un puerto hacia el borde se vuelva root):
```bash
interface g0/2
 spanning-tree guard root
```

Loop Guard (protege contra unidireccionalidad de BPDUs):
```bash
interface g0/1
 spanning-tree guard loop
```

Verificación:
```bash
show spanning-tree summary
show spanning-tree vlan 10
show spanning-tree interface g0/1 detail
```

Buenas prácticas:
- Active Rapid-PVST+ salvo necesidad de MST en dominios grandes.
- Aplique PortFast y BPDU Guard en todo puerto de acceso.
- Use Root Guard en enlaces hacia acceso cuando dist sea la raíz esperada en distribución/core.


## 7. EtherChannel: PAgP, LACP, estático; consistencia y verificación

Objetivo: agrupar varios enlaces físicos en un lógico (Port-Channel) para más ancho de banda y redundancia.

Modos:
- Está tico (on): sin negociación.
- PAgP (Cisco): desirable/auto.
- LACP (estándar): active/passive.

Requisitos de consistencia entre puertos miembros:
- Mismo modo duplex/velocidad, VLANs permitidas, tipo de puerto (access/trunk), nativa, etc.

Ejemplo con LACP en un trunk:
```bash
conf t
interface range g0/1 - 2
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 channel-group 1 mode active
!
interface port-channel 1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 description LAG-a-Distribucion
end
```

Con PAgP:
```bash
interface range g0/1 - 2
 channel-group 2 mode desirable
```

Estático (sin LACP/PAgP):
```bash
interface range g0/3 - 4
 channel-group 3 mode on
```

Balanceo de carga (global):
```bash
port-channel load-balance src-dst-ip
! opciones típicas: mac, ip, port
```

Verificación:
```bash
show etherchannel summary
show etherchannel port-channel
show lacp neighbor
show pagp neighbor
```

Buenas prácticas:
- Prefiera LACP (estándar) frente a PAgP (propietario).
- Configure siempre primero los puertos físicos coherentes y luego el Port-Channel.
- Documente qué VLANs pasan por cada Port-Channel.


## 8. Port-Security: sticky MAC, violaciones, aging e interacción con Voice VLAN

Objetivo: limitar dispositivos por puerto de acceso y mitigar ataques L2 básicos.

Configuración típica (1 MAC por puerto con sticky):
```bash
conf t
interface fa0/10
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict
 spanning-tree portfast
 spanning-tree bpduguard enable
 description Acceso-IP-Phone+PC
end
```

Modos de violación:
- protect: descarta en silencio; no log.
- restrict: descarta y loguea/contabiliza.
- shutdown: pone el puerto en err-disabled (más drástico).

Aging de direcciones seguras:
```bash
interface fa0/10
 switchport port-security aging time 5
 switchport port-security aging type inactivity
```

Verificación y limpieza:
```bash
show port-security interface fa0/10
show port-security address
clear port-security sticky interface fa0/10
```

Notas con Voice VLAN:
- Incrementar `maximum` para contemplar teléfono IP + PC (ej. 2).
- `switchport voice vlan` agrega una sub-encapsulación 802.1Q para voz.


## 9. Voice VLAN y QoS básica en acceso

Voice VLAN:
```bash
conf t
interface fa0/10
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 spanning-tree portfast
 mls qos trust cos
! o, en plataformas con auto QoS:
! auto qos voip cisco-phone
end
```

Consideraciones:
- Use `mls qos trust cos` o `trust dscp` según el diseño QoS.
- Verifique que el teléfono IP marque CoS/DSCP esperado.
- Asegure que la VLAN de voz tenga DHCP con opción 150 (según solución de telefonía).


## 10. Operación y diagnóstico: show/clear, errdisable, storm-control, CDP/LLDP

Comandos de estado generales:
```bash
show interfaces status
show interfaces counters errors
show interfaces switchport
show mac address-table dynamic
show vlan brief
show interfaces trunk
show spanning-tree summary
```

CDP/LLDP para descubrimiento:
```bash
show cdp neighbors detail
! lldp run
! show lldp neighbors detail
```

Storm control (mitigar broadcast/multicast storms):
```bash
interface fa0/10
 storm-control broadcast level 5.00 3.00
 storm-control multicast level 5.00 3.00
 storm-control action shutdown
```

Errdisable recovery:
```bash
show errdisable recovery
errdisable recovery cause bpduguard psecure-violation link-flap
errdisable recovery interval 300
```

Limpieza/operaciones:
```bash
clear mac address-table dynamic interface fa0/10
clear counters interface g0/1
```

Buenas prácticas:
- Activar storm-control en puertos de acceso sensibles.
- Registrar eventos críticos y habilitar recuperación controlada de errdisable cuando corresponda.


## 11. Plantillas rápidas por función (Access/Distribution)

Plantilla — Puerto de acceso solo datos:
```bash
interface range fa0/1 - 24
 description ACCESO-DATOS
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 spanning-tree bpduguard enable
 switchport nonegotiate
```

Plantilla — Puerto de acceso datos+voz:
```bash
interface range fa0/1 - 24
 description ACCESO-VOZ
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 spanning-tree portfast
 spanning-tree bpduguard enable
 mls qos trust cos
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict
```

Plantilla — Trunk a switch de distribución:
```bash
interface g0/1
 description TRUNK-A-DIST-1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 switchport trunk native vlan 999
 switchport nonegotiate
 spanning-tree guard loop
```

Plantilla — EtherChannel LACP (2 enlaces):
```bash
interface range g0/1 - 2
 description LACP-A-DIST-1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 channel-group 1 mode active
!
interface port-channel 1
 description LAG-A-DIST-1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
```


## 12. Checklist final de verificación

- VLANs creadas y nombradas correctamente: `show vlan brief`.
- Puertos de acceso en VLAN adecuada y con PortFast+BPDU Guard: `show interfaces switchport`.
- Trunks 802.1Q operativos, VLAN nativa dedicada y listas restringidas: `show interfaces trunk`.
- STP en Rapid-PVST+; root definido por VLAN y protecciones activas: `show spanning-tree summary`.
- EtherChannel formado con todos los miembros up: `show etherchannel summary`.
- Port-Security en acceso con límites y sticky correctos: `show port-security interface ...`.
- Voice VLAN operativa y QoS/trust aplicado en puertos con teléfonos.
- Storm-control configurado y errdisable recovery documentado.
- Gestión por SSH funcionando y logs/NTP coherentes (ver [ios/README.md](ios/README.md)).