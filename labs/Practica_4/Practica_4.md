# Práctica 4

## Objetivo de la práctica

- Revisar y dominar los comandos de verificación en routers y switches Cisco para identificar:
  - Estado y parámetros de interfaces físicas y lógicas (SVI)
  - Tabla de direcciones MAC
  - Direcciones IP configuradas y vecindario ARP

## Materiales necesarios

- Packet Tracer
- Archivo .pkt de esta práctica: Practica_4.pkt

## Descripción del escenario

- Diagrama

  ![Diagrama Practica 4](./Prac4.png)
- Lista de Dispositivos
  - Router
    - 4331
  - Switches
    - 2960
  - End Devices
    - PC (2)
- Direccionamiento IP
  - Utiliza el direccionamiento incluido en el .pkt. Si construyes el escenario desde cero, puedes usar 192.168.10.0/24 con gateway 192.168.10.254 y 2 PCs en la misma red.

## Requerimientos técnicos

- Ejecutar y documentar, en router y switch, los comandos para:
  - Ver el estado y detalles de interfaces
  - Ver y filtrar la tabla MAC
  - Ver direcciones IP, ARP y rutas conectadas
  - Descubrir vecinos con CDP/LLDP
- Generar tráfico para poblar tablas (pings entre hosts)
- Guardar evidencias de salida de comandos

---

## Procedimiento en el Switch (SW-A-01)

Antes de iniciar, genera tráfico entre los hosts (ping PC-1 -> PC-2) para que el switch aprenda MACs.

### Resumen rápido de interfaces

```bash
SW-A-01#show interfaces status
```

Output:

```bash
Port      Name               Status       Vlan       Duplex  Speed Type
Fa0/1                        notconnect   1          auto    auto  10/100BaseTX
Fa0/2                        notconnect   1          auto    auto  10/100BaseTX
Fa0/3                        notconnect   1          auto    auto  10/100BaseTX
Fa0/4                        notconnect   1          auto    auto  10/100BaseTX
Fa0/5                        notconnect   1          auto    auto  10/100BaseTX
Fa0/6                        notconnect   1          auto    auto  10/100BaseTX
Fa0/7                        notconnect   1          auto    auto  10/100BaseTX
Fa0/8                        notconnect   1          auto    auto  10/100BaseTX
Fa0/9                        notconnect   1          auto    auto  10/100BaseTX
Fa0/10                       connected    1          auto    auto  10/100BaseTX
Fa0/11                       connected    1          auto    auto  10/100BaseTX
Fa0/12                       notconnect   1          auto    auto  10/100BaseTX
Fa0/13                       notconnect   1          auto    auto  10/100BaseTX
Fa0/14                       notconnect   1          auto    auto  10/100BaseTX
Fa0/15                       notconnect   1          auto    auto  10/100BaseTX
Fa0/16                       notconnect   1          auto    auto  10/100BaseTX
Fa0/17                       notconnect   1          auto    auto  10/100BaseTX
Fa0/18                       notconnect   1          auto    auto  10/100BaseTX
Fa0/19                       notconnect   1          auto    auto  10/100BaseTX
Fa0/20                       notconnect   1          auto    auto  10/100BaseTX
Fa0/21                       notconnect   1          auto    auto  10/100BaseTX
Fa0/22                       notconnect   1          auto    auto  10/100BaseTX
Fa0/23                       notconnect   1          auto    auto  10/100BaseTX
Fa0/24                       notconnect   1          auto    auto  10/100BaseTX
Gig0/1                       connected    1          auto    auto  10/100BaseTX
Gig0/2                       notconnect   1          auto    auto  10/100BaseTX
```


```bash
SW-A-01#show ip interface brief
```

```bash
Interface              IP-Address      OK? Method Status                Protocol 
FastEthernet0/1        unassigned      YES manual down                  down 
FastEthernet0/2        unassigned      YES manual down                  down 
FastEthernet0/3        unassigned      YES manual down                  down 
FastEthernet0/4        unassigned      YES manual down                  down 
FastEthernet0/5        unassigned      YES manual down                  down 
FastEthernet0/6        unassigned      YES manual down                  down 
FastEthernet0/7        unassigned      YES manual down                  down 
FastEthernet0/8        unassigned      YES manual down                  down 
FastEthernet0/9        unassigned      YES manual down                  down 
FastEthernet0/10       unassigned      YES manual up                    up 
FastEthernet0/11       unassigned      YES manual up                    up 
FastEthernet0/12       unassigned      YES manual down                  down 
FastEthernet0/13       unassigned      YES manual down                  down 
FastEthernet0/14       unassigned      YES manual down                  down 
FastEthernet0/15       unassigned      YES manual down                  down 
FastEthernet0/16       unassigned      YES manual down                  down 
FastEthernet0/17       unassigned      YES manual down                  down 
FastEthernet0/18       unassigned      YES manual down                  down 
FastEthernet0/19       unassigned      YES manual down                  down 
FastEthernet0/20       unassigned      YES manual down                  down 
FastEthernet0/21       unassigned      YES manual down                  down 
FastEthernet0/22       unassigned      YES manual down                  down 
FastEthernet0/23       unassigned      YES manual down                  down 
FastEthernet0/24       unassigned      YES manual down                  down 
GigabitEthernet0/1     unassigned      YES manual up                    up 
GigabitEthernet0/2     unassigned      YES manual down                  down 
Vlan1                  unassigned      YES manual administratively down down
```


Notas:
- show interfaces status es muy útil para ver “connected/notconnect” y velocidad/duplex.
- show ip interface brief muestra IPs de SVIs (si existen) y estado de capa 2/3.

### VLANs y troncales

```bash
SW-A-01#show vlan brief
SW-A-01#show interfaces trunk
```

Output:

```bash
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Fa0/1, Fa0/2, Fa0/3, Fa0/4
                                                Fa0/5, Fa0/6, Fa0/7, Fa0/8
                                                Fa0/9, Fa0/10, Fa0/11, Fa0/12
                                                Fa0/13, Fa0/14, Fa0/15, Fa0/16
                                                Fa0/17, Fa0/18, Fa0/19, Fa0/20
                                                Fa0/21, Fa0/22, Fa0/23, Fa0/24
                                                Gig0/1, Gig0/2
1002 fddi-default                     active    
1003 token-ring-default               active    
1004 fddinet-default                  active    
1005 trnet-default                    active 
```

### Detalle de una interfaz

```bash
SW-A-01#show interfaces FastEthernet0/1
```

```bash
FastEthernet0/1 is down, line protocol is down (disabled)
  Hardware is Lance, address is 000a.417c.2101 (bia 000a.417c.2101)
 BW 100000 Kbit, DLY 1000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation ARPA, loopback not set
  Keepalive set (10 sec)
  Full-duplex, 100Mb/s
  input flow-control is off, output flow-control is off
  ARP type: ARPA, ARP Timeout 04:00:00
  Last input 00:00:08, output 00:00:05, output hang never
  Last clearing of "show interface" counters never
  Input queue: 0/75/0/0 (size/max/drops/flushes); Total output drops: 0
  Queueing strategy: fifo
  Output queue :0/40 (size/max)
  5 minute input rate 0 bits/sec, 0 packets/sec
  5 minute output rate 0 bits/sec, 0 packets/sec
     956 packets input, 193351 bytes, 0 no buffer
     Received 956 broadcasts, 0 runts, 0 giants, 0 throttles
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored, 0 abort
     0 watchdog, 0 multicast, 0 pause input
     0 input packets with dribble condition detected
     2357 packets output, 263570 bytes, 0 underruns
     0 output errors, 0 collisions, 10 interface resets
     0 babbles, 0 late collision, 0 deferred
     0 lost carrier, 0 no carrier
     0 output buffer failures, 0 output buffers swapped out
```

Puntos clave que verás:
- Contadores (input/output errors, drops)
- Velocidad y dúplex negociados
- Línea “Hardware is ... , address is xxxx.xxxx.xxxx” con la MAC del puerto
- Configuración aplicada (description, switchport mode, vlan, etc.)

### 4) Tabla de direcciones MAC (CAM)

```bash
SW-A-01#show mac address-table
```

Output:

```bash
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----

   1    0090.2be5.e602    DYNAMIC     Gig0/1
```
```bash
SW-A-01#show mac address-table dynamic
```

```bash
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----

   1    0090.2be5.e602    DYNAMIC      Gig0/1
```

Notas:
- Si no ves entradas, genera tráfico (ping) para que el switch aprenda MACs.
- Puedes limpiar entradas dinámicas con:

```bash
SW-A-01#clear mac address-table dynamic
```


### Descubrimiento de vecinos

```bash
SW-A-01#show cdp neighbors

```
```bash
Capability Codes: R - Router, T - Trans Bridge, B - Source Route Bridge
                  S - Switch, H - Host, I - IGMP, r - Repeater, P - Phone
Device ID    Local Intrfce   Holdtme    Capability   Platform    Port ID
RT-A-01      Gig 0/1          135            R       ISR4300     Gig 0/0/1
```

```bash
SW-A-01#show cdp neighbors detail
```
```bash
Device ID: RT-A-01
Entry address(es): 
  IP address : 192.168.100.254
Platform: cisco ISR4300, Capabilities: Router
Interface: GigabitEthernet0/1, Port ID (outgoing port): GigabitEthernet0/0/1
Holdtime: 121

Version :
Cisco IOS XE Software, Version 03.13.04.S - Extended Support Release
Cisco IOS Software, ISR Software (X86_64_LINUX_IOSD-UNIVERSALK9-M), Version 15.5(3)S5, RELEASE SOFTWARE (fc2)
Technical Support: http://www.cisco.com/techsupport
Copyright (c) 1986-2017 by Cisco Systems, Inc.
Compiled Mon 05-Oct-15 11:24 by mcpre

advertisement version: 2
Duplex: full
```

Tip:
- CDP viene habilitado por defecto en la mayoría de switches Cisco.
- LLDP puede requerir habilitarse:

## Procedimiento en el Router (RT-A-01)

Genera tráfico primero (pings a PCs) para poblar ARP y verificar conectividad.

### Vistazo rápido de interfaces

```bash
RT-A-01#show ip interface brief
```
```bash
Interface              IP-Address      OK? Method Status                Protocol 
GigabitEthernet0/0/0   unassigned      YES unset  up                    down 
GigabitEthernet0/0/1   192.168.100.254 YES manual up                    up 
GigabitEthernet0/0/2   unassigned      YES unset  up                    down 
Vlan1                  unassigned      YES unset  up                    down
```

```bash
RT-A-01#show protocols
```
```bash
Global values:
  Internet Protocol routing is enabled
GigabitEthernet0/0/0 is up, line protocol is down
GigabitEthernet0/0/1 is up, line protocol is up
  Internet address is 192.168.100.254/24
GigabitEthernet0/0/2 is up, line protocol is down
Vlan1 is up, line protocol is down
```

### 2) Detalle de una interfaz e IP

```bash
RT-A-01#show interfaces GigabitEthernet 0/0/1
```
```bash
GigabitEthernet0/0/1 is up, line protocol is up (connected)
  Hardware is ISR4331-3x1GE, address is 0090.2be5.e602 (bia 0090.2be5.e602)
  Internet address is 192.168.100.254/24
  MTU 1500 bytes, BW 1000000 Kbit, DLY 10 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation ARPA, loopback not set
  Keepalive not supported
  output flow-control is on, input flow-control is on
  ARP type: ARPA, ARP Timeout 04:00:00, 
  Last input 00:00:08, output 00:00:05, output hang never
  Last clearing of "show interface" counters never
  Input queue: 0/375/0 (size/max/drops); Total output drops: 0
  Queueing strategy: fifo
  Output queue :0/40 (size/max)
  5 minute input rate 13 bits/sec, 0 packets/sec
  5 minute output rate 13 bits/sec, 0 packets/sec
     8 packets input, 820 bytes, 0 no buffer
     Received 4 broadcasts (0 IP multicasts)
     0 runts, 0 giants, 0 throttles
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored
     0 watchdog, 1017 multicast, 0 pause input
     0 input packets with dribble condition detected
     8 packets output, 776 bytes, 0 underruns
     0 output errors, 0 collisions, 1 interface resets
     0 unknown protocol drops
     0 babbles, 0 late collision, 0 deferred
     0 lost carrier, 0 no carrier
     0 output buffer failures, 0 output buffers swapped out
```

```bash
RT-A-01#show ip interface GigabitEthernet0/0/1
```

```bash
GigabitEthernet0/0/1 is up, line protocol is up (connected)
  Internet address is 192.168.100.254/24
  Broadcast address is 255.255.255.255
  Address determined by setup command
  MTU is 1500 bytes
  Helper address is not set
  Directed broadcast forwarding is disabled
  Outgoing access list is not set
  Inbound  access list is not set
  Proxy ARP is enabled
  Security level is default
  Split horizon is enabled
  ICMP redirects are always sent
  ICMP unreachables are always sent
  ICMP mask replies are never sent
  IP fast switching is disabled
  IP fast switching on the same interface is disabled
  IP Flow switching is disabled
  IP Fast switching turbo vector
  IP multicast fast switching is disabled
  IP multicast distributed fast switching is disabled
  Router Discovery is disabled
  IP output packet accounting is disabled
  IP access violation accounting is disabled
  TCP/IP header compression is disabled
  RTP/IP header compression is disabled
  Probe proxy name replies are disabled
  Policy routing is disabled
  Network address translation is disabled
  BGP Policy Mapping is disabled
  Input features: MCI Check
  WCCP Redirect outbound is disabled
  WCCP Redirect inbound is disabled
  WCCP Redirect exclude is disabled
```


Observa:
- MAC del puerto (línea Hardware address ...)
- MTU, encapsulado, input/output errors
- ACLs o políticas aplicadas (si existieran)

### ARP e IPs aprendidas

```bash
RT-A-01#show arp
```
```bash
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  192.168.100.52          7   00E0.F9DD.4D42  ARPA   GigabitEthernet0/0/1
Internet  192.168.100.254         -   0090.2BE5.E602  ARPA   GigabitEthernet0/0/1
```
```bash
RT-A-01#show ip arp
```
```bash
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  192.168.100.52          7   00E0.F9DD.4D42  ARPA   GigabitEthernet0/0/1
Internet  192.168.100.254         -   0090.2BE5.E602  ARPA   GigabitEthernet0/0/1
```



### Rutas conectadas y reachability

```bash
RT-A-01#show ip route connected
```

```bash
C   192.168.100.0/24  is directly connected, GigabitEthernet0/0/1
```
```bash
RT-A-01#ping 192.168.100.52
```
```bash
Sending 5, 100-byte ICMP Echos to 192.168.100.52, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 0/0/0 ms
```
```bash
RT-A-01#traceroute 192.168.100.52
```
```bash

Type escape sequence to abort.
Tracing the route to 192.168.100.52

  1   192.168.100.52  0 msec    0 msec    0 msec    
```

### 5) Descubrimiento de vecinos

```bash
RT-A-01#show cdp neighbors
```
```bash
Capability Codes: R - Router, T - Trans Bridge, B - Source Route Bridge
                  S - Switch, H - Host, I - IGMP, r - Repeater, P - Phone
Device ID    Local Intrfce   Holdtme    Capability   Platform    Port ID
SW-A-01      Gig 0/0/1        158            S       2960        Gig 0/1
RT-A-01#show cdp  neighbors de
RT-A-01#show cdp  neighbors detail 
```

```bash
RT-A-01#show cdp neighbors detail
```
```bash
Device ID: SW-A-01
Entry address(es): 
Platform: cisco 2960, Capabilities: Switch
Interface: GigabitEthernet0/0/1, Port ID (outgoing port): GigabitEthernet0/1
Holdtime: 146

Version :
Cisco IOS Software, C2960 Software (C2960-LANBASEK9-M), Version 15.0(2)SE4, RELEASE SOFTWARE (fc1)
Technical Support: http://www.cisco.com/techsupport
Copyright (c) 1986-2013 by Cisco Systems, Inc.
Compiled Wed 26-Jun-13 02:49 by mnguyen

advertisement version: 2
Duplex: full
```

### IPv6 (opcional si el escenario lo usa)

```bash
RT-A-01#show ipv6 interface brief
```
```bash

GigabitEthernet0/0/0       [up/down]
    unassigned
GigabitEthernet0/0/1       [up/up]
    unassigned
GigabitEthernet0/0/2       [up/down]
    unassigned
Vlan1                      [up/down]
    unassigned
```
```bash
RT-A-01#show ipv6 neighbors
```

## Ejercicio guiado

1) Desde PC-1, hacer ping a PC-2 y al gateway.
2) En el switch:
   - show mac address-table
   - show mac address-table interface Fa0/1
3) En el router:
   - show ip interface brief
   - show arp
   - show ip route connected
4) Validar vecinos:
   - En ambos equipos: show cdp neighbors detail
5) Repite pings si no aparecen entradas (las tablas envejecen).

## Tabla de referencia rápida

Switch (2960):
- show interfaces status
- show interfaces description
- show ip interface brief
- show vlan brief
- show interfaces trunk
- show interfaces TYPE/NUM
- show running-config interface TYPE/NUM
- show mac address-table [dynamic|interface X|address MAC]
- clear mac address-table dynamic
- show arp
- show cdp neighbors [detail]
- show lldp neighbors [detail]

Router (4331):
- show ip interface brief
- show interfaces description
- show interfaces TYPE/NUM
- show ip interface TYPE/NUM
- show running-config interface TYPE/NUM
- show arp / show ip arp
- show ip route [connected]
- ping X.X.X.X
- traceroute X.X.X.X
- show cdp neighbors [detail]
- show lldp neighbors [detail]

---

## Buenas prácticas

- Genera tráfico (ping) antes de revisar MAC/ARP.
- Usa description en interfaces para facilitar identificación.
- Verifica dúplex/velocidad y errores físicos al diagnosticar.
- Guarda evidencia con copy run start antes de cerrar la práctica.
