# proyecto-protocolos-comunicacion
# 🚀 Proyecto: Automatización de Red con Ansible y RESTCONF

Este repositorio contiene la configuración y los scripts de Ansible para automatizar el despliegue del protocolo de enrutamiento OSPFv3 en una topología de red basada en Cisco CSR1000v. El objetivo es demostrar el uso de herramientas de Infraestructura como Código (IaC) para gestionar configuraciones de red complejas de manera eficiente, reproducible y escalable.

## 📖 Guía de Implementación Completa

Esta guía detalla todos los pasos necesarios para replicar el proyecto, desde la preparación del entorno hasta la ejecución de la automatización.

---

### **Parte 1: Preparación del Entorno**

En esta sección, se realiza la configuración base de los equipos de red y se prepara el nodo de control que orquestará la automatización.

#### **Fase 1: Configuración Manual de Routers**

Se aplica una configuración inicial a los routers para establecer conectividad y habilitar las interfaces de gestión (CLI y API RESTCONF).

##### **Paso 1: Configurar el Router Principal (`CSR1000v`)**

Este router es el objetivo principal de nuestra automatización.

1.  **Configuración Inicial (Hostname, Usuario y API RESTCONF):**
    *   Se establece el `hostname` para identificación.
    *   Se crea un usuario `ansible` con privilegios de nivel 15.
    *   Se habilita el servidor HTTPS y la interfaz `restconf` para la gestión vía API.

    ```sh
    enable
    configure terminal
    hostname CSR1000v
    ipv6 unicast-routing
    username ansible privilege 15 secret cisco123
    ip http secure-server
    restconf
    end
    wr
    ```

2.  **Configurar Interfaz de Gestión (`GigabitEthernet3`):**
    *   Esta interfaz conectará el router con la máquina Ubuntu (nodo de control Ansible). Se le asigna una dirección IPv6 estática.

    ```sh
    enable
    config t
    interface GigabitEthernet3
    ipv6 address 2001:DB8:ACAD:3::1/64
    ipv6 enable
    no shutdown
    exit
    end
    wr
    ```

3.  **Habilitar Login Local en Líneas VTY:**
    *   Permite la autenticación a través de la terminal virtual usando credenciales locales.

    ```sh
    enable
    configure terminal
    line vty 0 4
    login local
    end
    write memory
    ```

##### **Paso 2: Configurar el Router Vecino 1 (`R1-1`)**

*   Se configura su `hostname`, se activa el enrutamiento IPv6, se asigna una IP a la interfaz de conexión y se habilita OSPFv3 en ella con un `router-id` único.

```sh
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
```

##### **Paso 3: Configurar el Router Vecino 2 (`R1-2`)**

*   La configuración es análoga a la de `R1-1`, asegurando una identidad única en la red.

```sh
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
```

#### **Fase 2: Configuración del Nodo de Control (Ubuntu)**

Se prepara la máquina Linux desde la cual se lanzará la automatización.

##### **Paso 4: Configurar Red e Instalar Ansible**

1.  **Configuración de Red:**
    *   Asegúrese de que el nodo Ubuntu tenga una interfaz de red en el mismo segmento que la interfaz de gestión del `CSR1000v` (red `2001:DB8:ACAD:3::/64`).

2.  **Instalación de Ansible:**
    *   Con conectividad a internet, instale el motor de Ansible.

    ```sh
    sudo apt update
    sudo apt install ansible-core -y
    ```

3.  **Verificación de Conectividad:**
    *   Es **crucial** confirmar que el nodo de control puede alcanzar la interfaz de gestión del router principal.

    ```sh
    ping 2001:db8:acad:3::1
    ```

---

### **Parte 2: Automatización con Ansible**

Una vez preparado el entorno, procedemos a definir y ejecutar la lógica de automatización.

#### **Fase 3: Creación de Archivos de Automatización**

Estos archivos le indican a Ansible *qué* dispositivos gestionar y *qué* tareas ejecutar.

##### **Paso 5: Crear el Archivo de Inventario (`inventory.yml`)**

El inventario define los hosts y las variables de conexión. Este proyecto utiliza un enfoque híbrido, por lo que el inventario debe facilitar tanto conexiones API como CLI.

*   Cree el archivo `inventory.yml` en el directorio de su proyecto:

```yaml
all:
  hosts:
    CSR1000v:
      ansible_host: 2001:db8:acad:3::1
  vars:
    ansible_connection: httpapi
    ansible_httpapi_use_ssl: true
    ansible_httpapi_validate_certs: false
    ansible_httpapi_port: 443
    ansible_user: ansible
    ansible_password: cisco123
    ansible_network_os: restconf
    # Desactiva la verificación de clave de host SSH para las tareas que usan network_cli
    ansible_ssh_host_key_checking: false
```

##### **Paso 6: Crear el Playbook (`playbook_ospfv3.yml`)**

El playbook contiene la secuencia de tareas. Este playbook demuestra una técnica avanzada que combina dos tipos de conexión: `uri` para la API RESTCONF y `cisco.ios.ios_config` para la CLI.

*   Cree el archivo `playbook_ospfv3.yml`. La clave de este playbook es la capacidad de **sobrescribir el método de conexión por tarea** usando la sección `vars`, como se muestra en el ejemplo:

```yaml
# ... (tareas iniciales usando el módulo uri para RESTCONF) ...

- name: 4a. Habilitar OSPFv3 en GigabitEthernet1
  cisco.ios.ios_config:
    lines:
      - ipv6 ospf 1 area 0
    parents: interface GigabitEthernet1
  vars:
    # Sobrescribimos la conexión para esta tarea específica
    ansible_connection: network_cli
    ansible_network_os: cisco.ios.ios

# ... (más tareas) ...
```
> El contenido completo del playbook se encuentra en el archivo `playbook_ospfv3.yml` de este repositorio.

#### **Fase 4: Ejecución y Verificación**

El último paso es ejecutar la automatización y confirmar que el estado de la red es el esperado.

##### **Paso 7: Ejecutar el Playbook de Ansible**

*   Desde la terminal del nodo de control, en la carpeta del proyecto, lance el comando:

```sh
ansible-playbook -i inventory.yml playbook_ospfv3.yml
```

##### **Paso 8: Verificar el Resultado**

1.  **Salida de Ansible:** La última tarea (`Mostrar vecinos OSPF`) imprimirá una estructura de datos JSON con la información de los vecinos detectados.
2.  **Verificación Manual en el Router:** Para una confirmación definitiva, acceda al `CSR1000v` y ejecute:

    ```sh
    show ipv6 ospf neighbor
    show ipv6 route ospf
    show running-config
    ```
    La salida deberá mostrar las adyacencias OSPF en estado `FULL` y las rutas aprendidas.

---

**¡Felicidades!** Ha completado con éxito la configuración y automatización de una red utilizando Ansible, combinando la potencia de las APIs modernas con la flexibilidad de la CLI tradicional.
