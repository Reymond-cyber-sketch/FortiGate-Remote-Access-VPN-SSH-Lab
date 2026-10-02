# FortiGate-Remote-Access-VPN-SSH-Lab

Laboratorio de VPN Remote Access con FortiGate en GNS3, acceso HTTPS al servidor sin necesidad de VPN y acceso SSH únicamente mediante un túnel IPsec.

## Video de demostración

Video del funcionamiento completo del laboratorio:

https://youtu.be/zSZO28Cchb4

---

## Descripción

En este laboratorio se implementó una infraestructura de red utilizando FortiGate, MikroTik, GNS3 y contenedores Docker.

El objetivo principal fue permitir que un usuario pudiera acceder al servicio HTTPS de un servidor web sin necesidad de conectarse a una VPN, mientras que el acceso administrativo mediante SSH solamente estuviera disponible después de establecer una VPN Remote Access con el FortiGate.

Para representar un escenario más realista también se utilizó un MikroTik como router de los usuarios y otro MikroTik para simular el proveedor de Internet.

Durante las pruebas se comprobó que el servicio HTTPS permanecía disponible con la VPN desconectada, mientras que el acceso SSH al servidor requería que el usuario se conectara previamente al túnel VPN.

---

## Objetivo del laboratorio

Configurar una infraestructura donde:

- El usuario pueda acceder al servidor web mediante HTTPS sin utilizar VPN.
- El usuario pueda acceder al servidor mediante SSH únicamente después de conectarse a una VPN Remote Access.
- El FortiGate funcione como firewall y servidor VPN.
- El MikroTik USER-ROUTER proporcione DHCP y NAT a la red de usuarios.
- Otro MikroTik funcione como ISP.
- La red de usuarios utilice VLAN 10.
- El servidor se encuentre dentro de una red /28.
- Se pueda realizar traceroute hacia el servidor.

---

## Topología

La topología utilizada fue la siguiente:

```text
USER-PC
   |
SW-LAB
   |
USER-ROUTER
   |
   ISP
   |
FGT-USER
   |
WEB-SERVER
```

El ISP también se encuentra conectado al nodo NAT de GNS3 para proporcionar salida a Internet.

El FortiGate posee adicionalmente una conexión de administración mediante su interfaz port3.

---

## Funcionamiento general

La red fue diseñada para manejar dos tipos de acceso diferentes.

### Acceso HTTPS

El usuario puede acceder al servidor web utilizando la dirección pública del FortiGate:

```text
https://198.51.100.98
```

El FortiGate recibe la conexión en el puerto TCP 443 y la redirige hacia:

```text
10.24.96.130:443
```

Este acceso funciona sin necesidad de establecer la VPN.

### Acceso SSH

El servidor también tiene habilitado SSH, pero este servicio no fue publicado directamente hacia Internet.

Para utilizar SSH, el USER-PC debe establecer primero la VPN Remote Access contra el FortiGate.

Después de conectarse, el cliente recibe una dirección IP virtual y puede acceder al servidor privado mediante:

```text
ssh sshuser@10.24.96.130
```

---

## Direccionamiento IP

### Red de usuarios

```text
Red:
10.24.96.0/25

Gateway:
10.24.96.1

DHCP:
10.24.96.10 - 10.24.96.120

VLAN:
10
```

Durante las pruebas el USER-PC recibió:

```text
10.24.96.11/25
```

---

### Red del servidor

```text
Red:
10.24.96.128/28

Gateway:
10.24.96.129

WEB-SERVER:
10.24.96.130
```

---

### Red entre ISP y FortiGate

```text
Red:
198.51.100.96/30

ISP:
198.51.100.97

FGT-USER:
198.51.100.98
```

---

### Red entre ISP y USER-ROUTER

```text
Red:
203.0.113.96/30

ISP:
203.0.113.97

USER-ROUTER:
203.0.113.98
```

---

### Red de administración del FortiGate

```text
FGT-USER port3:
192.168.178.2/24
```

---

### Pool de clientes VPN

```text
10.24.97.10 - 10.24.97.20
```

Durante las pruebas el USER-PC recibió:

```text
10.24.97.10
```

---

## Configuración del FortiGate

El FortiGate funciona como firewall principal, gateway de la red del servidor y servidor de VPN Remote Access.

### Interfaces

```text
port1 - WAN
198.51.100.98/30

port2 - WEB-LAN
10.24.96.129/28

port3 - MGMT
192.168.178.2/24
```

---

## Red del servidor

La interfaz port2 del FortiGate funciona como gateway del WEB-SERVER.

```text
FortiGate:
10.24.96.129/28

WEB-SERVER:
10.24.96.130/28

Gateway del servidor:
10.24.96.129
```

---

## VPN Remote Access

La VPN fue configurada en el FortiGate con el nombre:

```text
REMOTE-SSH
```

Características principales:

```text
Tipo:
Remote Access / Dial-Up

IKE:
IKEv1

Mode:
Main Mode

XAuth:
Enabled

Mode Config:
Enabled

Encryption:
DES

Authentication:
SHA256

DH Group:
14

Pool VPN:
10.24.97.10 - 10.24.97.20
```

El usuario utilizado para la VPN fue:

```text
vpnuser
```

El grupo configurado fue:

```text
VPN-USERS
```

Las contraseñas y la clave precompartida no fueron publicadas en este repositorio.

---

## Phase 2

La segunda fase del túnel utiliza:

```text
Nombre:
REMOTE-SSH-P2

Proposal:
DES-SHA256

PFS:
Disabled
```

La configuración permite que los clientes Remote Access establezcan dinámicamente sus selectores.

---

## Publicación HTTPS

Para permitir acceso HTTPS sin VPN se creó un Virtual IP en el FortiGate.

```text
Nombre:
VIP_HTTPS_WEB

IP pública:
198.51.100.98

Puerto externo:
443

IP interna:
10.24.96.130

Puerto interno:
443

Protocolo:
TCP
```

---

## Política HTTPS

Se configuró una política para permitir el acceso desde Internet hacia el servidor web.

```text
Nombre:
WAN_TO_WEB_HTTPS

Origen:
port1

Destino:
port2

Servicio:
HTTPS

Destino:
VIP_HTTPS_WEB
```

Esta política permite que el USER-PC acceda al servidor mediante:

```text
https://198.51.100.98
```

sin necesidad de conectarse a la VPN.

---

## Política SSH mediante VPN

El acceso SSH solamente se permite desde la interfaz VPN hacia el WEB-SERVER.

```text
Nombre:
VPN_TO_WEB_SSH

Origen:
REMOTE-SSH

Destino:
port2

Source Address:
VPN-CLIENT-POOL

Destination Address:
WEB-SERVER-HOST

Servicio:
SSH

Puerto:
22
```

De esta forma el servidor SSH no queda publicado directamente hacia Internet.

---

## USER-ROUTER

El MikroTik USER-ROUTER funciona como gateway de la red de usuarios.

### Interfaces

```text
ether1 - WAN
203.0.113.98/30

ether2 - LAN USERS
10.24.96.1/25
```

---

## DHCP

El USER-ROUTER entrega direcciones IP a los clientes de la red.

```text
Pool:
10.24.96.10 - 10.24.96.120

Gateway:
10.24.96.1

DNS:
8.8.8.8
1.1.1.1
```

---

## NAT del USER-ROUTER

La red:

```text
10.24.96.0/25
```

utiliza NAT masquerade hacia:

```text
ether1
```

para poder acceder al ISP y a Internet.

---

## Configuración del ISP

El ISP fue simulado utilizando otro MikroTik.

Sus interfaces principales fueron:

```text
Hacia FGT-USER:
198.51.100.97/30

Hacia USER-ROUTER:
203.0.113.97/30

Hacia GNS3 NAT:
DHCP Client
```

El ISP utiliza masquerade hacia la interfaz conectada al nodo NAT de GNS3.

Su función es permitir comunicación entre ambas redes públicas y proporcionar salida a Internet.

---

## VLAN de usuarios

La red de usuarios utiliza:

```text
VLAN 10
```

Los puertos principales del switch quedaron asociados a la red de usuarios.

```text
USER-PC
|
SW-LAB
|
USER-ROUTER
```

El FortiGate utiliza una conexión separada para administración mediante la red MGMT.

---

## WEB-SERVER

El servidor fue configurado con:

```text
IP:
10.24.96.130/28

Gateway:
10.24.96.129
```

Servicios utilizados:

```text
HTTPS
TCP 443

SSH
TCP 22
```

---

## Servicio HTTPS

El servidor web responde mediante HTTPS.

Prueba realizada:

```bash
curl -k https://198.51.100.98/
```

Resultado:

```text
WEB SERVER - Laboratorio FortiGate

Servidor HTTPS funcionando correctamente.

IP: 10.24.96.130
```

Esta prueba fue realizada sin VPN.

---

## Servicio SSH

También se configuró SSH en el WEB-SERVER.

Usuario utilizado:

```text
sshuser
```

Dirección utilizada:

```text
10.24.96.130
```

El acceso se realiza mediante:

```bash
ssh sshuser@10.24.96.130
```

---

## Prueba de VPN Remote Access

Desde USER-PC se estableció la VPN hacia el FortiGate.

Durante la negociación se obtuvo:

```text
XAuth authentication of 'vpnuser' successful
```

El FortiGate asignó al cliente la IP virtual:

```text
10.24.97.10
```

También se estableció correctamente el túnel IPsec:

```text
CHILD_SA REMOTE-SSH established
connection 'REMOTE-SSH' established successfully
```

---

## Prueba SSH con VPN

Con la VPN activa se realizó:

```bash
ssh sshuser@10.24.96.130
```

La autenticación fue exitosa y se obtuvo acceso al servidor.

```text
Welcome to Alpine!

WEB-SERVER:~$
```

Esto confirmó que el USER-PC podía administrar el servidor mediante SSH a través de la VPN.

---

## Prueba SSH sin VPN

Después de desconectar la VPN se volvió a intentar acceder mediante:

```bash
ssh sshuser@10.24.96.130
```

La comunicación no fue posible.

Esto confirmó que el acceso SSH depende de la VPN Remote Access.

---

## Prueba HTTPS sin VPN

Con la VPN desconectada se volvió a probar:

```bash
curl -k https://198.51.100.98/
```

El servidor respondió correctamente.

Esto confirmó que HTTPS no depende de la VPN.

---

## Traceroute

También se realizó una prueba de traceroute desde el USER-PC hacia el servidor.

```bash
traceroute 10.24.96.130
```

Esta prueba permitió observar el camino utilizado para alcanzar la red del servidor cuando la VPN se encontraba activa.

---

## Resultado final

El comportamiento obtenido fue:

```text
SIN VPN

HTTPS:
FUNCIONA

SSH:
NO FUNCIONA
```

```text
CON VPN

HTTPS:
FUNCIONA

SSH:
FUNCIONA
```

De esta forma el servicio web se encuentra disponible para los usuarios externos, mientras que el acceso administrativo mediante SSH permanece protegido por la VPN.

---

## Estructura del repositorio

```text
FortiGate-Remote-Access-VPN-SSH-Lab/
|
|-- README.md
|
|-- configs/
|   |-- FGT-REMOTE-ACCESS.txt
|   |-- USER-ROUTER-MikroTik.txt
|   |-- ISP-MikroTik.txt
|
|-- images/
|   |-- 01-topologia.png
|   |-- 02-fortigate-interfaces.png
|   |-- 03-fortigate-vpn-remote-active.png
|   |-- 04-user-pc-vpn-connected.png
|   |-- 05-https-sin-vpn.png
|   |-- 06-ssh-con-vpn.png
|   |-- 07-ssh-sin-vpn-falla.png
|   |-- 08-traceroute-servidor.png
|
|-- scripts/
    |-- comandos-pruebas.txt
```

---

## Archivos de configuración

En la carpeta `configs/` se encuentran los principales datos utilizados para configurar:

```text
FGT-REMOTE-ACCESS.txt
USER-ROUTER-MikroTik.txt
ISP-MikroTik.txt
```

Las contraseñas y claves utilizadas en la VPN fueron reemplazadas por valores ocultos antes de publicar el repositorio.

---

## Conclusión

En este laboratorio se implementó correctamente una VPN Remote Access utilizando FortiGate y un cliente strongSwan dentro de GNS3.

La infraestructura permitió separar el acceso público al servidor web del acceso administrativo.

El servicio HTTPS pudo ser utilizado sin necesidad de establecer una VPN, mientras que para acceder mediante SSH fue necesario conectarse previamente al túnel IPsec.

También se implementaron DHCP, NAT, VLAN, direccionamiento público y privado, un ISP simulado y un servidor con servicios HTTPS y SSH.

Las pruebas realizadas permitieron comprobar que el comportamiento de la infraestructura cumplía con el objetivo planteado.

---

## Autor

Reymond Daniel Guerrero Cruz

Matrícula: 2024-0963
