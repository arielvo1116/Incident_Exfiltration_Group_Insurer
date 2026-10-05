# INCIDENTE DE EXFILTRACIÓN EN GRUPO ASEGURADOR

## RESUMEN Y CONTENIDO GENERAL

### RESUMEN DEL CASO
El pasado lunes se detecto una transferencia masiva de datos no autorizada a las 3:00 de la mañana, el tráfico sale de un servidor interno hacía un destino en Europa, este informe busca dar conocimiento sobre la situación a directores CISO y perfiles de seguridad de la información y ciberseguridad.

### CONTENIDO GENERAL
En esta situación se esta realizando una auditoria sobre la situación presentada para poder determinar el origen de dicho problema, soluciones y planes de acción para evitar futuros ataques.

#### ANALISIS DE ARQUITECTURA
El equipo de infraestructura dividio la red en 2, utilizando una mascara de red /25, lo que da como resultado do subredes.

- Subred 1: LAN Empleados
- Subred 2: DMZ Servidores

En este caso marcamos los limites exactos de la subred 2 de la siguiente manera:

IPs Asignadas: 172.16.50.128 - 172.16.50.255

** IP de identificación: 172.16.50.128
** IP de Broadcast: 172.16.50.255
** Rango de IP asignables disponibles: 172.16.50.129 - 172.16.50.254

La IP comprometida por el ataque es la IP "172.16.50.150" por lo cual es perteneciente a la red DMZ de servidores, por lo que la red LAN de Empleados no se verá afectada ya que nada saltara a esta red

#### CAZA DE AMENAZAS
Se ejecuto en la linea de comandos de linux el comando "ss -tuln" el cuál respondio con la siguiente línea:
tcp    ESTAB      0      0      172.16.50.150:58942       82.102.15.99:443

Esta linea nos indica que la información viajo desde el puerto -58942- hacía el puerto -443-, el puerto -443- es un puerto a la escucha, es un puerto fijo que siempre esta activo esperando a recibir información, mientras que el puerto -58942- es un puerto efímero que manda información hacia el puerto a la escucha, por lo que el malware no fue pensado para entrar, dicho malware se ejecuto desde dentro para enviar información hacia afuera 

Adicional, para identificar en que punto de internet se esta alojando el servidor del atacante con una dirección IP "82.102.15.99" se utilizo la linea de comando "traceroute 82.102.15.99" para poder identificar el camino exacto que recorrieron los paquetes, evitando utilizar "ping" ya que usando ese comando solamente veríamos el tiempo que tardan en llegar los paquetes y si hay paquetes que se perdieron en el camino

#### AUDITORIA DE REGLAS FIREWALL
Se reviso la configuración del firewall para identificar por que razón el atacante logro extraer los datos, y se encontraron las siguientes reglas aplicadas a la subred 2 DMZ Servidores en el siguiente orden

Regla 1: Acción: Permitir | Origen: DMZ | Destino: LAN | Puerto: 3306 (MySQL)

Regla 2: Acción: Permitir | Origen: DMZ | Destino: WAN | Puerto: Todos

Regla 3: Acción: Denegar  | Origen: Cualquiera | Destino: Cualquiera | Puerto: Todos (Implicit Deny)

Aquí encontramos el error, en la linea dos se le permitio a todos los puertos mandar información del DMZ a la red WAN y por esa razón el puerto -58942- logro viajar hacia el puerto -443- en la red. Se reescribe la regla 2 de la siguiente manera

Regla 2: Acción: Permitir | Origen: DMZ | Destino: WAN | Puerto: 443 (HTTPS)

#### PROPUESTA DE AUTOMATIZACIÓN
Se propone una automatización en python, programada en Ubuntu a través de Crontab para que se ejecute cada 5 minutos y lea la salida del comando "ss -tuln" y enviar una alerta automatica si detecta que el servidor de base de datos esta utilizando puertos de salida hacia internet.

Se crearia el script programado en python y se alojaria en un servidor que se mantenga siempre activo para que pueda ejecutarse a toda hora, incluso de madrugada sin necesidad de mantener ninguna computadora encendida
