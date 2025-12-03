# Routers Cisco — Guía magistral Cisco 1

Este documento cubre configuración básica, direccionamiento IPv4/IPv6, rutas estáticas, RIPv2, OSPFv2/OSPFv3, EIGRP (Named), NAT/PAT, DHCP, NTP, SSH y Syslog. Cada tema incluye comandos, explicación y verificación.

## Índice
- 1. Configuración base del router (plantilla segura)
- 2. Direccionamiento en interfaces (IPv4/IPv6) y verificación
- 3. Rutas estáticas y por defecto
- 4. RIPv2 (IPv4)
- 5. OSPFv2 (IPv4) y OSPFv3 (IPv6)
- 6. EIGRP Named Mode (IPv4)
- 7. NAT y PAT (overload)
- 8. Servidor DHCP (IPv4)
- 9. NTP, SSH, y Syslog

## 1. Configuración base del router (plantilla segura)

Objetivo: asegurar acceso, trazabilidad y consistencia operativa.

```bash
hostname R1
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
```

Buenas prácticas:
- Banners legales: banner motd/login (ver IOS/README).
- Limitar VTY a SSH y considerar ACL con access-class.
- Sincronizar hora (sección NTP) para logs coherentes.

Guardar cambios y activar versionado local:
```bash
archive
 path flash:configs/
 maximum 14
 write-memory
end
write memory
```

## 2. Direccionamiento en interfaces (IPv4/IPv6) y verificación

IPv4 en G0/0:
```bash
conf t
interface g0/0
 description LAN Usuarios
 ip address 192.168.10.1 255.255.255.0
 no shutdown
end
```

IPv6 en G0/1 con SLAAC y dirección estática:
```bash
conf t
ipv6 unicast-routing
interface g0/1
 description Enlace WAN
 ipv6 address 2001:db8:10:1::1/64
 ipv6 address autoconfig
 no shutdown
end
```

Verificación rápida:
```bash
show ip interface brief
show ipv6 interface brief
show interfaces description
ping 192.168.10.10
ping 2001:db8:10:1::10
```

## 3. Rutas estáticas y por defecto

Ruta estática a red remota (IPv4):
```bash
ip route 10.20.0.0 255.255.0.0 192.0.2.2
```

Ruta por siguiente salto vs. por interfaz de salida:
```bash
ip route 10.30.0.0 255.255.0.0 192.0.2.2
ip route 10.40.0.0 255.255.0.0 g0/1
```

Ruta por defecto (IPv4 e IPv6):
```bash
ip route 0.0.0.0 0.0.0.0 192.0.2.1
ipv6 route ::/0 2001:db8:ffff::1
```

Rutas flotantes (mayor distancia administrativa):
```bash
ip route 10.20.0.0 255.255.0.0 192.0.2.2 1
ip route 10.20.0.0 255.255.0.0 198.51.100.2 5
```

Verificación:
```bash
show ip route static
show ipv6 route static
traceroute 10.20.1.10
```

## 4. RIPv2 (IPv4)

Activación y mejores prácticas:
```bash
conf t
router rip
 version 2
 no auto-summary
 passive-interface default
 no passive-interface g0/0
 no passive-interface g0/1
 network 192.168.10.0
 network 198.51.100.0
 exit
```

Autenticación MD5 (opcional):
```bash
key chain RIP-KEYS
 key 1
  key-string ClaveMD5Segura
!
interface g0/1
 ip rip authentication mode md5
 ip rip authentication key-chain RIP-KEYS
```

Verificación:
```bash
show ip protocols
show ip rip database
show ip route rip
debug ip rip
```

## 5. OSPFv2 (IPv4) y OSPFv3 (IPv6)

OSPFv2 de una sola área (area 0):
```bash
conf t
router ospf 1
 router-id 1.1.1.1
 passive-interface default
 no passive-interface g0/0
 no passive-interface g0/1
 network 192.168.10.0 0.0.0.255 area 0
 network 198.51.100.0 0.0.0.3 area 0
 exit
```

Ajustes por interfaz (métrica/costo, tipo de red):
```bash
interface g0/1
 ip ospf cost 100
 ip ospf network point-to-point
```

OSPFv3 (IPv6) clásico en IOS:
```bash
conf t
ipv6 unicast-routing
ipv6 router ospf 10
 router-id 1.1.1.1
!
interface g0/0
 ipv6 ospf 10 area 0
!
interface g0/1
 ipv6 ospf 10 area 0
```

Verificación:
```bash
show ip ospf neighbor
show ip ospf interface brief
show ip route ospf
show ipv6 ospf neighbor
show ipv6 route ospf
```

## 6. EIGRP Named Mode (IPv4)

Activación (ASN 100, nombre EIGRP-CORE):
```bash
conf t
router eigrp EIGRP-CORE
 !
 address-family ipv4 autonomous-system 100
  af-interface default
   passive-interface
  exit-af-interface
  af-interface g0/1
   no passive-interface
  exit-af-interface
  network 192.168.10.0 0.0.0.255
  network 198.51.100.0 0.0.0.3
  metric weights 0 1 0 1 0 0
  eigrp stub connected summary
 exit-address-family
```

Notas:
- Named Mode centraliza IPv4/IPv6 bajo un “address-family”.
- passive-interface por defecto; habilite sólo donde hay vecinos.
- metric weights por defecto 0 1 0 1 0 0 equivale a K1=1, K3=1.

Autenticación HMAC-SHA (si IOS lo soporta) o MD5:
```bash
key chain EIGRP-KEYS
 key 1
  key-string ClaveFuerte
!
interface g0/1
 ip authentication mode eigrp 100 md5
 ip authentication key-chain eigrp 100 EIGRP-KEYS
```

Verificación:
```bash
show ip eigrp neighbors
show ip eigrp topology
show ip route eigrp
```

## 7. NAT y PAT (overload)

Zonas inside/outside:
```bash
conf t
interface g0/0
 ip address 192.168.10.1 255.255.255.0
 ip nat inside
!
interface g0/1
 ip address 198.51.100.2 255.255.255.252
 ip nat outside
```

PAT usando la interfaz de salida (típico hogar/pyme):
```bash
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat inside source list 1 interface g0/1 overload
```

NAT dinámico con pool:
```bash
ip nat pool NAT-POOL 198.51.100.100 198.51.100.110 netmask 255.255.255.0
access-list 10 permit 192.168.10.0 0.0.0.255
ip nat inside source list 10 pool NAT-POOL overload
```

NAT estático 1:1 para servidor interno:
```bash
ip nat inside source static 192.168.10.50 198.51.100.50
```

Verificación:
```bash
show ip nat translations
show ip nat statistics
clear ip nat translations *
```

## 8. Servidor DHCP (IPv4)

Reservas y exclusiones:
```bash
conf t
ip dhcp excluded-address 192.168.10.1 192.168.10.20
ip dhcp excluded-address 192.168.10.200 192.168.10.254
```

Pool para la VLAN/segmento:
```bash
ip dhcp pool LAN-USUARIOS
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8 1.1.1.1
 domain-name ejemplo.local
 lease 7
```

DHCP Relay (si el servidor está en otra red):
```bash
interface g0/0
 ip helper-address 192.168.100.10
```

Verificación:
```bash
show ip dhcp binding
show ip dhcp pool
debug ip dhcp server events
```

## 9. NTP, SSH y Syslog

NTP — hora consistente:
```bash
conf t
clock timezone CST -6 0
clock summer-time CDT recurring
ntp server 192.0.2.10 prefer
ntp server 198.51.100.10
ntp update-calendar
end
show clock detail
show ntp associations
```

SSH — endurecido:
```bash
conf t
ip domain-name ejemplo.local
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 2
crypto key generate rsa modulus 2048
line vty 0 4
 transport input ssh
 login local
 exec-timeout 10 0
exit
```

Syslog — centralización de logs:
```bash
conf t
logging buffered 16384 debugging
logging trap informational
logging host 192.0.2.50
logging source-interface g0/1
service timestamps log datetime msec localtime show-timezone
no logging console
end
show logging
```

Checklist final de verificación:
- show ip interface brief / show ipv6 interface brief sin administratively down.
- ping y traceroute a redes claves exitosos.
- Tablas de routing muestran rutas esperadas (static, rip, ospf, eigrp).
- NAT/PAT generando translations bajo carga.
- Clientes obtienen IP por DHCP y reachability hacia Internet/cores.
- NTP sincronizado, SSH accesible, y logs visibles en el servidor Syslog.