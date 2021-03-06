---

copyright:
  years: 2017
lastupdated: "2017-10-12"

---

{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:codeblock: .codeblock}
{:pre: .pre}
{:screen: .screen}
{:tip: .tip}
{:download: .download}

# Gestionar cortafuegos
Virtual Router Appliance (VRA) tiene la capacidad de procesar reglas de cortafuegos para proteger las VLAN direccionadas a través del dispositivo. Los cortafuegos de VRA pueden dividirse en dos pasos: 

1. Definición de uno o más conjuntos de reglas 
2. Aplicación de un conjunto de reglas para una interfaz o zona. Una zona consiste en una o más interfaces de red.

Es importante probar las reglas de cortafuegos después de la creación para garantizar que las reglas funcionan según lo previsto y que las nuevas reglas no restringen el acceso administrativo al dispositivo. 

Durante la manipulación de reglas en la interfaz `dp0bond1`, se recomienda conectar con el dispositivo utilizando `dp0bond0`. La conexión a la consola mediante Intelligent Platform Management Interface (IPMI) también es una opción. 

## Sin estado vs con estado 
De forma predeterminada, el cortafuegos es sin estado, pero puede configurarse como con estado en caso de que sea necesario. Un cortafuegos sin estado necesitará reglas para el tráfico en ambas direcciones, mientras que los cortafuegos con estado se utilizan para conexiones en las que solo se requieren reglas de tráfico de entrada. Para configurar un cortafuegos con estado, debe especificar conexiones establecidas o relacionadas para el tráfico de salida. 

Para que todos los cortafuegos sean "con estado", ejecute el mandato siguiente: 

```
set security firewall global-state-policy icmp
set security firewall global-state-policy tcp
set security firewall global-state-policy udp
```

Para que los cortafuegos individuales sean "con estado": 

```
set security firewall name TEST rule 1 allow
set security firewall name TEST rule 1 state enable
```

## Conjunto de reglas de cortafuegos 
Las reglas de cortafuegos se agrupan en conjuntos con nombre para facilitar la aplicación de regla en varias interfaces. Cada conjunto de reglas tiene una acción predeterminada asociada. Considere el ejemplo siguiente:
```
set security firewall name ALLOW_LEGACY default-action accept
set security firewall name ALLOW_LEGACY rule 1 action drop
set security firewall name ALLOW_LEGACY rule 1 source address network-group1 set security firewall name ALLOW_LEGACY rule 2 action drop set security firewall name ALLOW_LEGACY rule 2 destination port 23 set security firewall name ALLOW_LEGACY rule 2 log set security firewall name ALLOW_LEGACY rule 2 protocol tcp set security firewall name ALLOW_LEGACY rule 2 source address network-group2
```

En el conjunto de reglas, `ALLOW_LEGACY`, hay dos reglas definidas. La primera regla descarta cualquier tráfico que proviene de un grupo de dirección denominado `network-group1`. La segunda regla descarta y registra cualquier tráfico destinado para el puerto telnet (`tcp/23`) del grupo de dirección denominado `network-group2`. La acción predeterminada indica que se acepta cualquier otra acción. 

## Cómo permitir el acceso de centro de datos 
IBM ofrece varias subredes IP para proporcionar servicios y soporte a los sistemas que se ejecutan en el centro de datos. Por ejemplo, los servicios de resolución de DNS se ejecutan en `10.0.80.11` y `10.0.80.12`. Otras subredes se utilizan durante el suministro y soporte.Encontrará los rangos de IP utilizados en los centros de datos en este [artículo](http://knowledgelayer.softlayer.com/faq/what-ip-ranges-do-i-allow-through-firewall).

^^Tenemos que cambiar el enlace .. KnowledgeLayer desaparecerá ^^
^^ solo cuando la misma información de seguimiento esté en un documento de Bluemix o en otro lugar. Lo arreglaremos más tarde ^^

Puede permite el acceso de centro de datos colocando las reglas `SERVICE-ALLOW` adecuadas al principio de los conjuntos de reglas de cortafuegos con la acción `accept`. El lugar en el que el conjunto de reglas debe aplicarse depende del diseño de cortafuegos y direccionamiento implementado.

Se recomienda colocar reglas de cortafuegos en la ubicación que causa la duplicación de trabajo. Por ejemplo, permitir la entrada de subredes de back-end en `dp0bond0` será menos trabajo que permitir la salida de subredes de back-end en cada interfaz virtual de VLAN.

### Reglas de cortafuegos por interfaz 
Un método para configurar el cortafuegos en VRA es aplicar conjuntos de reglas de cortafuegos en cada interfaz. En este caso, una interfaz puede ser de plano de datos (`dp0s0`) o virtual (`dp0bond0.303`). Cada interfaz tiene tres asignaciones de cortafuegos posibles: 

`in`  El cortafuegos se comprueba en paquetes que entran mediante la interfaz. Los paquetes se pueden atravesar o destinar para VRA.

`out`  El cortafuegos se comprueba en paquetes que salen mediante la interfaz. Estos paquetes pueden estar atravesando VRA o pueden estar originados en VRA.

`local`  El cortafuegos se comprueba en paquetes que están destinados directamente en VRA. 

Una interfaz puede tener varios conjuntos de reglas aplicados en cada dirección. Se aplican por orden de configuración. Tenga en cuenta que esto no es posible en el tráfico de cortafuegos que proviene de un dispositivo VRA que utiliza cortafuegos por interfaz.

A modo de ejemplo, para asignar el conjunto de reglas `ALLOW_LEGACY` a la opción `in` para la interfaz `bp0s1`, utilice el mandato de configuración siguiente: 

`set interfaces dataplane dp0s1 firewall in ALLOW_LEGACY `

## Control Plane Policing (CPP)
Control plane policing (CPP) proporciona protección contra los ataques en Virtual Router Appliance y le permite configurar políticas de cortafuegos asignadas a las interfaces deseadas y aplicar dichas políticas a los paquetes que entran y salen de VRA. 

CPP se implementa cuando la palabra clave `local` se utiliza en las políticas de cortafuegos que se asignan a cualquier tipo de interfaz VRA, como las interfaces de plano de datos o bucle de retorno. A diferencia de las reglas de cortafuegos aplicadas a los paquetes que atraviesan VRA, la acción predeterminada de las reglas de cortafuegos para el plano de control de entrada y salida de tráfico es `Allow`.  Los usuarios deben añadir reglas de descarte explícitas si el comportamiento predeterminado no es el deseado. 

VRA proporciona un conjunto de regla CPP básica como plantilla. Puede fusionarlo con su configuración ejecutando: 

`vyatta@vrouter# merge /opt/vyatta/etc/cpp.conf`

Después de fusionar este conjunto de regla, se añade y se aplica un nuevo conjunto de reglas de cortafuegos denominado `CPP` en la interfaz loopback. Se recomienda modificar este conjunto de reglas para que se adapte a su entorno. 

## Cortafuegos basados en zonas
Otro concepto de cortafuegos de Virtual Router Appliance son los cortafuegos basados en zonas. En una operación de cortafuegos basado en zonas, se asigna una interfaz a una zona (solo una zona por interfaz) y los conjuntos de reglas de cortafuegos se asignan a los límites entre las zonas, con la idea de que todas las interfaces de una zona tengan el mismo nivel de seguridad y permiso para direccionar libremente. El tráfico solo se analiza cuando pasa de una zona a otra. Las zonas descartan cualquier tráfico que llega que no esté permitido de manera explícita. 

Una interfaz puede pertenecer a una zona o tener una configuración de cortafuegos por interfaz; pero no puede realizar ambas. 

Imagínese el siguiente caso de ejemplo de oficina con tres departamentos, cada uno con su propia VLAN: 

* Departamento A - VLAN 10 y 20 (interfaces dp0bond1.10 y dp0bond1.20)
* Departamento B - VLAN 30 y 40 (interfaces dp0bond1.30 y dp0bond1.40)
* Departamento C - VLAN 50 (interfaz dp0bond1.50)

Se puede crear una zona para cada departamento y las interfaces de dicho departamento pueden añadirse a la zona. El ejemplo siguiente lo ilustra: 
```
set security zone-policy zone DEPARTMENTA interface dp0bond1.10
set security zone-policy zone DEPARTMENTA interface dp0bond1.20  set security zone-policy zone DEPARTMENTB interface dp0bond1.30  set security zone-policy zone DEPARTMENTB interface dp0bond1.40  set security zone-policy zone DEPARTMENTC interface dp0bond1.50
```

El mandato `commit` rellena cada zona como una interfaz y las reglas de descarte predeterminadas descartan el tráfico que intenta entrar en la zona desde el exterior. En el ejemplo, VLAN 10 y 20 pueden pasar tráfico porque están en la misma zona (`DEPARTMENTA`), pero VLAN 10 y VLAN 30 no pueden pasar el tráfico porque VLAN 30 está en una zona distinta (`DEPARTMENTB`).

Las interfaces de cada zona pueden pasar el tráfico libremente y las reglas pueden definirse para la interacción entre zonas. Un conjunto de reglas se configura desde el punto de vista de dejar una zona a otra. El mandato siguiente muestra un ejemplo de cómo configurar una regla: 

`set security zone-policy zone DEPARTMENTC to DEPARTMENTB firewall ALLOW_PING `

Este mandato asocia la transición de DEPARTMENTC a DEPARTMENTB con el conjunto de reglas denominado `ALLOW_PING`. El tráfico que entra en la zona DEPARTMENTB desde la zona DEPARTMENTC se comprueba en este conjunto de reglas. 

Es importante comprender que no existe ninguna sentencia que declare que esta asignación de la zona DEPARTMENTC entrando en la zona DEPARTMENTB se pueda realizar a la inversa. Si no hay reglas que permitan el tráfico de la zona DEPARTMENTB a la zona DEPARTMENTC, el tráfico (respuestas de ICMP) no volverá a los hosts de DEPARTMENTC. 

## Registro de la sesión y el paquete 
VRA admite dos tipos de registros: 
1. Registro de sesión.   Utilice el mandato ``security firewall session-log`` para configurar el registro de sesión del cortafuegos.
  
  En UDP, ICMP, y todos los flujos que no sean de TCP, una sesión realiza una transición a cuatro estados durante el tiempo de vida del flujo. Para cada transición, puede configurar VRA para que registre un mensaje. TCP tiene un mayor número de transiciones de estado y cada una puede configurarse para realizar un registro.   

*	Registro por paquete. Incluya la palabra clave ``log`` en el cortafuegos o la regla NAT para registrar cada paquete de red que coincida con la regla.

  El registro por paquete se produce en las vías de acceso de reenvío de paquetes y genera grandes cantidades de salidas. Puede reducir significativamente el rendimiento de VRA y aumentar considerablemente el espacio de disco utilizado para los archivos de registro. Se recomienda utilizar el registro de paquete SOLO para fines de depuración. Para fines operativos, debe utilizarse el registro de sesión con estado. 
