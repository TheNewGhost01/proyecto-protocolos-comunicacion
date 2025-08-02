# proyecto-protocolos-comunicacion
# 🚀 Proyecto: Automatización de Red con Ansible y RESTCONF

Este repositorio contiene la configuración y los scripts de Ansible para automatizar el despliegue del protocolo de enrutamiento OSPFv3 en una topología de red basada en Cisco CSR1000v. El objetivo es demostrar el uso de herramientas de Infraestructura como Código (IaC) para gestionar configuraciones de red complejas de manera eficiente, reproducible y escalable.

## 📖 Guía de Implementación: Parte 1 - Preparación del Entorno

Esta guía detalla los pasos necesarios para preparar el entorno de red manualmente antes de ejecutar la automatización con Ansible. Se divide en dos fases principales: la configuración de los equipos de red y la preparación del nodo de control.

---

### **Fase 1: Configuración Manual de Routers**

En esta fase, se realiza la configuración base de los routers para establecer la conectividad inicial y habilitar las interfaces de gestión necesarias para que Ansible pueda comunicarse a través de CLI y la API RESTCONF.

#### **Paso 1: Configurar el Router Principal (`CSR1000v`)**

Este router es el objetivo principal de nuestra automatización.

1.  **Configuración Inicial (Hostname, Usuario y API RESTCONF):**
    *   Se establece el `hostname` para identificación.
    *   Se crea un usuario `ansible` con privilegios de nivel 15.
    *   Se habilita el servidor HTTPS y la interfaz `restconf` para la gestión vía API.

    ```bash
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

    ```bash
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
    *   Esto permite la autenticación a través de la terminal virtual usando el usuario y contraseña locales.

    ```bash
    enable
    configure terminal
    line vty 0 4
    login local
    end
    write memory
    ```

#### **Paso 2: Configurar el Router Vecino 1 (`R1-1`)**

Este es uno de los routers con los que `CSR1000v` establecerá adyacencia OSPF.

*   Se configura su `hostname`, se activa el enrutamiento IPv6, se asigna una IP a la interfaz de conexión y se habilita OSPFv3 en ella con un `router-id` único.

```bash
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

#### **Paso 3: Configurar el Router Vecino 2 (`R1-2`)**

La configuración es análoga a la de `R1-1`, asegurando una identidad única en la red.

*   Se repite el proceso de configuración de `hostname`, IPv6, interfaz y OSPFv3, asignando un `router-id` distinto.

```bash
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

---

### **Fase 2: Configuración del Nodo de Control (Ubuntu)**

Se prepara la máquina Linux desde la cual se orquestará la automatización.

#### **Paso 4: Configurar Red, Conectividad e Instalar Ansible**

1.  **Configuración de Red:**
    *   Asegúrese de que el nodo Ubuntu tenga una interfaz de red configurada en el mismo segmento que la interfaz de gestión del `CSR1000v` (red `2001:DB8:ACAD:3::/64`).

2.  **Instalación de Ansible:**
    *   Con la conectividad a internet funcionando, instale el motor de Ansible.

    ```bash
    sudo apt update
    sudo apt install ansible-core -y
    ```

3.  **Verificación de Conectividad:**
    *   Es **crucial** confirmar que el nodo de control puede alcanzar la interfaz de gestión del router principal antes de continuar.

    ```bash
    ping 2001:db8:acad:3::1
    ```
