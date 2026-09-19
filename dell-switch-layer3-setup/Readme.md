
# Configuración Inicial y Enrutamiento L3 en Switch Dell Networking N1524P

**Autores:**

>M. en T.A. José Fausto Romero Lujambio

>Ing. Rodrigo Rodriguez Galindo

**Entorno Operativo:** Arch Linux (Kernel 7.0.12) / Consola Serial / PC del Director de la UDI  
**Hardware:** Switch Dell Networking N1524P (System Version 6.8.1.9)

## Descripción
Memoria técnica del proceso de configuración inicial, aseguramiento (hardening) y aprovisionamiento de un switch Dell N1524P. La configuración se realizó inicialmente fuera de banda (Out-of-Band) mediante una conexión serial empleando `picocom` desde un entorno Arch Linux, para posteriormente habilitar la gestión remota segura vía SSH, segmentación por VLANs y el enrutamiento de Capa 3 (Inter-VLAN routing).

## Objetivo
Desplegar una configuración base segura minimizando vectores de ataque (desactivando Telnet, HTTP y telemetría de fábrica), generar nuevas llaves criptográficas para SSH, segmentar la red en múltiples VLANs y habilitar las capacidades de router L3 del equipo.

---

## 1. Entorno de Conexión y Arranque

Para establecer la conexión física por consola desde Arch Linux, se utilizó la herramienta de comunicación serial `picocom` emulando el comportamiento de HyperTerminal[cite: 2].

```bash
# Ejecutado en la terminal de Arch Linux
picocom -b 9600 /dev/ttyUSB0

```

> **Explicación:** Inicia la conexión serial a través de la interfaz USB a una velocidad de 9600 baudios, sin paridad y con 8 bits de datos.
> 
> 

**Secuencia de Arranque (Boot Menu):**
Durante el encendido del switch, se interrumpió la secuencia para acceder al menú de arranque:

1. `display boot menu`
2. `option 10`
3. `option 1` (Continuar con el arranque normal).

---

## 2. Acceso Privilegiado y Desactivación de Telemetría

Una vez que el switch cargó el sistema operativo, se ingresó al modo privilegiado y se desactivó el software de rastreo/asistencia de Dell.

```text
enable
configure terminal
eula-consent support-assist reject

```

> **Explicación:** Ingresa al modo de configuración global y rechaza los términos de SupportAssist, desactivando efectivamente el "spyware" o telemetría de Dell para entornos de alta privacidad.

---

## 3. Aseguramiento del Equipo (Hardening) y Acceso Remoto

Se minimizaron los vectores de ataque desactivando protocolos en texto plano, borrando llaves antiguas y forzando el uso de SSH con criptografía asimétrica.

```text
no ip http server
no ip http secure-server

```

> **Explicación:** Desactiva la interfaz de administración web (HTTP/HTTPS) para minimizar la superficie de ataque, forzando la administración exclusiva por línea de comandos (CLI).

```text
ip telnet server disable

```

> **Explicación:** Apaga el servidor Telnet, el cual es altamente inseguro por transmitir datos (incluyendo contraseñas) en texto plano.

```text
crypto key zeroize dsa
crypto key zeroize rsa

```

> **Explicación:** Elimina (reinicia) los certificados de raíces de confianza y llaves criptográficas preexistentes o de fábrica para evitar vulnerabilidades heredadas.

```text
crypto key generate dsa
crypto key generate rsa

```

> **Explicación:** Genera nuevas llaves criptográficas seguras que se utilizarán para el intercambio de claves (Diffie-Hellman) y el túnel seguro de SSH.

```text
ip ssh server
ip ssh server algorithm encryption aes128-gcm

```

> **Explicación:** Habilita el servidor SSH y especifica el uso de algoritmos de cifrado modernos y seguros (como GCM) para la administración remota.

```text
username admin password Temporal1 privilege 15

```

> **Explicación:** Crea una cuenta de administrador local con el nivel máximo de privilegios (15) para permitir la gestión a través de la red una vez conectado por SSH.

---

## 4. Habilitación de Capa 3 (Enrutamiento)

Paso crítico para que el switch deje de operar exclusivamente en Capa 2 y comience a procesar tráfico IP entre diferentes redes.

```text
ip routing

```

> **Explicación:** Activa el motor de enrutamiento del switch. Esta es la instrucción clave que permite que el equipo L2 actúe como un router L3 y pueda intercomunicar las distintas VLANs (Inter-VLAN routing).

---

## 5. Segmentación Lógica (VLANs) y Asignación de Puertos

Creación de las redes virtuales y asignación de los puertos físicos del switch (tanto en rangos como de manera puntual).

```text
vlan 10,20
exit

```

> **Explicación:** Crea de manera simultánea las VLANs 10 y 20 en la base de datos del equipo.

```text
interface range gi1/0/1-12
switchport access vlan 10
exit

```

> **Explicación:** Selecciona un rango continuo de puertos (del 1 al 12) y los asigna como puertos de acceso dedicados a la VLAN 10.

```text
interface range gi1/0/13-23
switchport access vlan 20
exit

```

> **Explicación:** Selecciona el siguiente bloque de puertos (del 13 al 23) y los asigna a la VLAN 20.

```text
interface range gi1/0/1,gi1/0/3
switchport access vlan 1
exit

```

> **Explicación:** Ejemplo de selección de puertos puntuales o separados (sin usar un guion). Reasigna los puertos 1 y 3 de vuelta a la VLAN 1 (VLAN por defecto).

---

## 6. Configuración de Interfaz Virtual (SVI)

Asignación de una dirección IP lógica al switch para permitir la administración remota y actuar como puerta de enlace (Gateway) para los dispositivos de la red.

```text
interface vlan 10
ip address 172.18.0.1 255.255.255.240
exit

```

> **Explicación:** Crea la Interfaz Virtual del Switch (SVI) para la VLAN 10 y le asigna una dirección IP con una máscara de subred /28. *(Nota: Inicialmente se intentó asignar como `/28` directo, se removió con `no ip address` y se reescribió correctamente usando la notación decimal `255.255.255.240`)*.

---

## 7. Comandos de Verificación

Instrucciones utilizadas durante y después de la configuración para validar que los cambios se aplicaron correctamente.

```text
show running-config

```

> **Explicación:** Muestra toda la configuración activa actualmente en la memoria RAM del equipo.

```text
show ip interface

```

> **Explicación:** (Equivalente al `show ip-config` anotado). Despliega un resumen de las interfaces lógicas (SVI) y los puertos ruteados con sus respectivas direcciones IP.

```text
show interfaces status

```

> **Explicación:** Muestra una tabla con el estado físico de los puertos, velocidad negociada y a qué VLAN están asignados actualmente.

## Observaciones
Se comprobó que el aislamiento entre VLANs distintas impedía la comunicación inicial entre dos PCs. Al configurar las SVIs y ejecutar ip routing, el switch activó su motor de enrutamiento interno, permitiendo el tráfico Inter-VLAN de forma exitosa. Adicionalmente, el proceso consolidó una administración remota segura tras deshabilitar protocolos vulnerables y generar llaves criptográficas para SSH.
