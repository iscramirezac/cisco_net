# Cisco IOS — Fundamentos, modos, ayuda, archivos, AAA, backups y mejores prácticas

Este documento guía práctica para dominar Cisco IOS (Inter‑Network Operating System) a nivel CCNA/Cisco 1. Incluye conceptos clave, comandos explicados y secuencias recomendadas para una operación segura y repetible.


## 1. ¿Qué es Cisco IOS?

Cisco IOS es el sistema operativo de la mayoría de routers y switches Cisco. Provee:
- Plano de control: protocolos de enrutamiento, STP, etc.
- Plano de administración: CLI, SSH, logging, AAA.
- Plano de reenvío: tablas FIB/CAM para mover paquetes.

Arquitectura de memorias:
- ROM: bootstrap y diagnóstico.
- Flash: imagen del IOS y otros archivos.
- NVRAM: startup-config (config de arranque).
- RAM: running-config y tablas operativas.


## 2. Modos de operación y navegación
Prompt y significado:
```bash
- Router> Modo EXEC usuario (básico, sólo ver).
- Router# Modo EXEC privilegiado (administración).
- Router(config)# Modo de configuración global.
- Router(config-if)# Configuración de interfaz.
- Router(config-line)# Configuración de líneas (console/VTY).
- Router(config-router)# Configuración de procesos de routing.
```
Cambios de modo (secuencias más usadas):

```bash
enable
configure terminal
interface g0/0
description Enlace a SW-Access-1
ip address 192.168.10.1 255.255.255.0
no shutdown
end
write memory
```

Explicación:
- enable: eleva privilegios (requiere enable secret).
- configure terminal: entra a configuración global.
- interface g0/0: selecciona la interfaz.
- description/ip address/no shutdown: parametriza la interfaz.
- end: regresa a EXEC privilegiado.
- write memory o copy running-config startup-config: guarda en NVRAM.

Atajos de navegación:
- end o Ctrl+Z: salir a EXEC privilegiado.
- exit: subir un nivel en el modo actual.
- do <comando>: ejecutar un show sin salir del modo de configuración.


## 3. Ayuda contextual, edición y paginación

Ayuda sensible al contexto:
- ?: lista comandos disponibles en el contexto actual.
- show ?: lista subcomandos/show disponibles.
- clock s?: autocompleta prefijos válidos (tab completa).

Abreviaturas y autocompletar:
- Los comandos pueden abreviarse mientras sigan siendo únicos.
- Tab completa el token actual; ? muestra opciones sin ejecutar.

Historial y edición de línea:
```bash
terminal history size 256
show history
```

- Ctrl+A / Ctrl+E: ir al inicio/fin de la línea.
- Ctrl+U: borrar desde el cursor al inicio.
- Ctrl+W: borrar la palabra anterior.
- Ctrl+P / Ctrl+N: navegar historial previo/siguiente.

Paginación en salidas largas:
terminal length 0

- terminal length 0: deshabilita paginación (útil para copiar/pegar o TFTP).
- terminal length 24: restaura el valor por defecto 24 líneas.


## 4. Archivos, guardado y arranque

Ver archivos y versión:
```bash
show version
show file systems
dir flash:
```

Verificar integridad de imágenes/archivos:
```bash
verify /md5 flash:imagen.bin
```

Guardar y restaurar configuración:
```bash
copy running-config startup-config
copy startup-config running-config
copy running-config tftp:
copy tftp: running-config
```
Guardado automático y versionado con archive:
```bash
archive
 path flash:configs/
 maximum 14
 write-memory
archive config
show archive
```
- archive/write-memory: cada write memory crea un snapshot.
- maximum: cantidad de versiones retenidas.
- archive config rollback <n>: revertir a una versión previa.

Parámetros de arranque:
```bash
show boot
show romvar
show register
conf t
 boot system flash:imagen.bin
 exit
config-register 0x2102
```
- boot system: prioriza la imagen a iniciar.
- config-register 0x2102: arranque normal con startup-config.

Banners:
```bash
banner motd ^ Autorización restringida ^ 
banner login ^ Acceso sólo personal autorizado ^ 
banner exec ^ Bienvenido ^ 
```

## 5. Configuración básica segura (plantilla)

```bash
hostname R1
no ip domain-lookup
service timestamps debug datetime msec localtime show-timezone
service timestamps log datetime msec localtime show-timezone
service password-encryption
ip domain-name ejemplo.local

enable secret <SecretoFuerte>
username admin privilege 15 secret <SecretoFuerte>

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

interface g0/0
 description LAN de usuarios
 ip address 192.168.10.1 255.255.255.0
 no shutdown
exit

do write memory
```
Notas de seguridad:
- Prefiera enable secret y username … secret sobre contraseñas en texto claro.
- service password-encryption es ofuscación básica; no sustituye a secrets.
- Restrinja VTY a SSH únicamente (transport input ssh) y considere ACL con access-class.
- Configure banners legales (motd/login) y tiempos de inactividad razonables (exec-timeout).

- aaa new-model: habilita el framework AAA.
- authentication login default local: usa base local para autenticación.
- authorization exec default local: crea sesión EXEC con privilegios autorizados.
- En despliegues empresariales, sustituya “local” por RADIUS/TACACS+.


## 6. SSH y transferencia segura (SCP)

Requisitos para SSH:
- Definir ip domain-name y hostname.
- Generar claves RSA.
- Forzar versión 2.

ip domain-name ejemplo.local
```bash
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 2
```

Habilitar servidor SCP para copias seguras:
```bash
ip scp server enable
```
Ejemplos de copia:
```bash
copy running-config tftp:
copy running-config scp:
```
Sugerencias:
- Prefiera SCP/SFTP frente a TFTP en redes reales.
- Limite acceso de gestión con ACL y VRF de administración si está disponible.


## 7. Buenas prácticas operativas

- Documente: use description en interfaces y un esquema de nombres consistente.
- Estándares: homologue banners, timeouts, SSH, logging y rutas de archivo archive.
- Cambios controlados: terminal length 0 al exportar, luego restaure a 24.
- Verificación antes/después: show running-config section <objeto>, show ip interface brief.
- Deshabilite servicios innecesarios y CDP en enlaces externos si no aporta valor.
- Sincronice hora (NTP) y zona horaria para logs correlacionables.
- Respaldos: establezca política de snapshots con archive y copias off-box (SCP).
- Seguridad: no utilice enable password ni Telnet; limite intentos de login.


## 8. Apéndice — Atajos útiles

- Ctrl+Z: finalizar modo de configuración.
- Ctrl+A / Ctrl+E: inicio/fin de línea.
- Ctrl+U / Ctrl+W: borrar a inicio / borrar palabra.
- Tab: autocompletar; ?: ayuda contextual.
- show running-config | section <texto>: filtra secciones relevantes.
- show <comando> | include/exclude/begin <patrón>: filtrado de salida.


## 9. Verificación rápida (checklist)

- Acceso seguro por SSH habilitado y probado.
- Usuario local con secret y privilege adecuados.
- Config guardada y snapshot activo (archive show).
- Descripciones en interfaces críticas.
- Banners y timestamps consistentes.
- Paginación restaurada a 24 líneas.