# proyecto-protocolos-comunicacion

#Guía Completa: Configuración y Automatización de Red con OSPFv3 y Ansible (Versión Original)

Esta guía respeta íntegramente cada comando y archivo de tu configuración original, dividiéndose en cuatro partes:
Parte 1: Configuración Manual de Routers: Preparación inicial de los routers de la topología.
Parte 2: Configuración del Nodo de Control (Ubuntu): Preparación de la máquina que ejecutará Ansible.
Parte 3: Creación de Archivos Ansible Originales: Escritura del inventario y el playbook tal como los proporcionaste.
Parte 4: Ejecución y Verificación: Lanzamiento del playbook de Ansible y confirmación del resultado.
Parte 1: Configuración Manual de Routers
(Esta parte es idéntica a la anterior, ya que se basa en tus configuraciones iniciales que son correctas).

Paso 1: Configurar el Router Principal (CSR1000v)
Configuración inicial (hostname, usuario, RESTCONF):

bash
enable
configure terminal
hostname CSR1000v
ipv6 unicast-routing
username ansible privilege 15 secret cisco123
ip http secure-server
restconf
end
wr
Configura la interfaz de gestión (GigabitEthernet3):

bash
enable
config t
interface GigabitEthernet3
ipv6 address 2001:DB8:ACAD:3::1/64
ipv6 enable
no shutdown
exit
end
wr
Habilita el login local en las líneas VTY:

bash
enable
configure terminal
line vty 0 4
login local
end
write memory
Paso 2: Configurar el Router 1 (R1-1)
bash
enable
conf t
hostname R1-1
ipv6 unicast-routing
interface GigabitEthernet0/0
ipv6 enable
ipv6 address 2001:1::2/64
ipv6 ospf 1 area 0
no shutdown
exit
ipv6 router ospf 1
router-id 2.2.2.2
exit
end
write Memory
Paso 3: Configurar el Router 2 (R1-2)
bash
enable
conf t
hostname R1-2
ipv6 unicast-routing
interface GigabitEthernet0/0
no shutdown
ipv6 enable
ipv6 address 2001:2::2/64
ipv6 ospf 1 area 0
exit
ipv6 router ospf 1
router-id 3.3.3.3
exit
end
wr
Parte 2: Configuración del Nodo de Control (Ubuntu)
(Esta parte también se mantiene sin cambios).

Paso 4: Configurar la Red y Conectividad en Ubuntu
Asegúrate de que la red de tu Ubuntu pueda alcanzar al router en 2001:db8:acad:3::1.
Instala Ansible: sudo apt install ansible-core.
Verifica la conectividad: ping 2001:db8:acad:3::1.
Parte 3: Creación de Archivos Ansible Originales
Aquí usamos tus archivos originales, explicando cómo funcionan.

Paso 5: Crear el Archivo de Inventario (inventory.yml)
Este archivo define la conexión principal al CSR1000v a través de la API RESTCONF.

Crea el archivo: nano inventory.yml
Pega este contenido exacto:

inventory.yml
v3
all:
  hosts:
    CSR1000v:
      ansible_host: 2001:db8:acad:3::1
  vars:
    ansible_connection: httpapi
Explicación:
ansible_connection: httpapi: Le dice a Ansible que se conecte usando la API web del router, no SSH.
ansible_network_os: restconf: Especifica que el "sabor" de la API es RESTCONF.
ansible_ssh_host_key_checking: false: Aunque la conexión principal es httpapi, esta línea es útil porque tu playbook más adelante cambia temporalmente la conexión a network_cli (que usa SSH), y esto previene que Ansible se detenga a preguntar si confía en la clave SSH del host.
Paso 6: Crear el Playbook (playbook_ospfv3.yml)
Este playbook es interesante porque combina dos métodos de conexión: RESTCONF (uri) para la mayoría de las tareas y CLI (cisco.ios.ios_config) para habilitar OSPF en las interfaces.

Crea el archivo: nano playbook_ospfv3.yml
Pega tu playbook original:

playbook_ospfv3.yml
v2
- name: Configurar y Activar CSR1000v paso a paso
  hosts: CSR1000v
  gather_facts: no
 
  tasks:
 
Explicación Clave:
Pasos 1-3: Usan el módulo uri para enviar llamadas directas a la API RESTCONF del router. Esto es útil para configuraciones que se mapean bien al modelo de datos YANG.
Paso 4: Para habilitar OSPF en las interfaces, el playbook cambia de estrategia. Usa el módulo cisco.ios.ios_config y, lo más importante, dentro de cada tarea (vars:), sobrescribe la conexión a network_cli. Ansible se conectará por SSH para estas tres tareas específicas, ejecutará los comandos y luego volverá a usar httpapi para las tareas siguientes.
Paso 5: Vuelve a usar el módulo uri para obtener datos operativos, demostrando la flexibilidad de combinar ambos métodos.
Parte 4: Ejecución y Verificación
Paso 7: Ejecutar el Playbook de Ansible
Con los archivos originales en su sitio, el comando de ejecución es el mismo.

bash
ansible-playbook -i inventory.yml playbook_ospfv3.yml
