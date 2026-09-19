
# Configuración de Capa 2 y Administración de Switch Enterasys/Extreme Networks

**Autores:** 
M. en T.A. José Fausto Romero Lujambio y Ing. Rodrigo Rodriguez Galindo  
**Ubicación:** Unidad de Informática de la Escuela Superior de Cómputo (ESCOM - IPN)

## Descripción
Este documento contiene la memoria técnica y el script de configuración inicial aplicado a algunos switches de la capa de acceso (equipos Enterasys / Extreme Networks) dentro de la infraestructura de red de la ESCOM. El script abarca la asignación de VLANs de datos y administración, aseguramiento de puertos troncales, y configuración de protocolos de capa 2 para optimizar el rendimiento y la seguridad del tráfico.

## Objetivo
Establecer un estándar de configuración seguro y eficiente para los equipos de capa de acceso, garantizando la correcta segmentación del tráfico mediante VLANs, la mitigación de bucles a través de RSTP, y el aseguramiento del acceso administrativo remoto exclusivo por SSH.

---

## Despliegue de Comandos

A continuación, se detalla línea por línea el script de configuración utilizado, agrupado por su función dentro del equipo:

### 1. Gestión de Búfer y Tráfico (Buffer Bloat)
Optimización del manejo de paquetes para evitar cuellos de botella y saturación en la red.

```text
set system enhancedbuffermode enable
<yes>

```

> **Explicación:** Activa el modo de búfer mejorado del switch (confirmando con `<yes>`) para optimizar el manejo de colas de tráfico en entornos de alta demanda.

```text
set flowcontrol disable

```

> **Explicación:** Deshabilita el control de flujo (802.3x) a nivel de puerto físico, delegando la gestión de la congestión a los protocolos de capas superiores (como TCP).

### 2. Seguridad de Acceso Local

Limpieza de cuentas genéricas para evitar accesos no autorizados al sistema operativo del switch.

```text
show system login
clear system login ro
clear system login rw
show system login

```

> **Explicación:** Muestra las cuentas actuales, elimina las credenciales predeterminadas de fábrica de "solo lectura" (`ro`) y "lectura/escritura" (`rw`), y finalmente verifica que hayan sido borradas correctamente.

### 3. Creación y Asignación de VLANs

Segmentación lógica de la red para usuarios (56), administración (999) y otros servicios (2000).

```text
set vlan create 56,999,2000

```

> **Explicación:** Instancia las VLANs 56, 999 y 2000 en la base de datos local del switch.

```text
set port vlan fe.*.* 56 modify-egress

```

> **Explicación:** Asigna todos los puertos de acceso FastEthernet (`fe.*.*`) a la VLAN 56, configurándolos para enviar el tráfico sin etiqueta (untagged) hacia los dispositivos finales.

```text
set vlan egress 56,999,2000 ge.*.* tagged

```

> **Explicación:** Configura los puertos Gigabit (`ge.*.*`) como enlaces troncales, permitiendo el paso de las VLANs 56, 999 y 2000 con etiqueta 802.1Q hacia el switch de distribución o core.

```text
clear vlan egress 1 ge.*.*

```

> **Explicación:** Elimina la VLAN 1 (VLAN nativa por defecto) de los enlaces Gigabit por buenas prácticas de seguridad.

```text
set port ingress-filter *.*.* enable

```

> **Explicación:** Habilita el filtrado de ingreso en todos los puertos, forzando al switch a descartar paquetes que provengan de VLANs a las que el puerto receptor no pertenece.

```text
show vlan portinfo
save config 

```

> **Explicación:** Imprime un resumen de la configuración de puertos/VLANs para revisión del administrador y guarda los cambios en la memoria no volátil.

### 4. Administración Remota (VLAN de Gestión)

Configuración de la interfaz lógica y protocolos de acceso seguro.

```text
set host vlan 999
set ip address 10.204.56.235 mask 255.255.255.0 gateway 10.204.56.254
ping 10.204.56.254

```

> **Explicación:** Asigna la VLAN 999 como la interfaz de gestión, establece la dirección IP estática, la máscara de subred y la puerta de enlace predeterminada, seguido de una prueba de conectividad (`ping`) al gateway.

```text
set telnet disable all
set ssh enable

```

> **Explicación:** Apaga el servicio Telnet (inseguro/texto plano) y habilita el servidor SSH para cifrar las conexiones de administración remota.

```text
set prompt ESCOM-3001-CEC-SW235

```

> **Explicación:** Define el nombre de host (hostname) del equipo en la línea de comandos para facilitar su identificación visual.

### 5. Protocolos de Capa 2 y Descubrimiento

Mitigación de bucles lógicos y mapeo de la topología.

```text
show spantree version
set spantree version rstp

```

> **Explicación:** Verifica la versión actual de Spanning Tree y la cambia a Rapid Spanning Tree Protocol (RSTP - 802.1w) para asegurar una convergencia de red mucho más rápida en caso de caídas de enlaces.

```text
show neighbors

```

> **Explicación:** Muestra los dispositivos de red adyacentes conectados directamente a los puertos del switch mediante protocolos de descubrimiento (LLDP/EDP).

### 6. Consideraciones de Hardware (Retorno a Valores de Fábrica)

```text
set switch stack-port ethernet

```

> **Explicación:** *Nota Operativa:* Este comando convierte los puertos de apilamiento (stacking) traseros del equipo en puertos Ethernet convencionales. Como este proceso altera fundamentalmente la arquitectura de hardware que el sistema operativo reconoce, fuerza un reinicio del equipo y borra toda la configuración previa, actuando en la práctica como un reseteo de fábrica.
