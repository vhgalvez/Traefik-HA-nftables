# Traefik-HA-nftables

Este proyecto configura un conjunto de reglas de firewall utilizando `nftables` para un entorno de alta disponibilidad (HA) con Traefik y Kubernetes.

## Estructura del Proyecto

El proyecto está organizado de la siguiente manera:

```plaintext
LICENSE
README.md
ansible/
files/
    nftables.conf
inventory/
    hosts.ini
playbooks/
    setup_nftables.yml
```

- **`ansible/`**: Contiene configuraciones adicionales para Ansible.
- **`files/nftables.conf`**: Archivo de configuración de `nftables` con las reglas necesarias para el entorno.
- **`inventory/hosts.ini`**: Archivo de inventario para definir los nodos de destino.
- **`playbooks/setup_nftables.yml`**: Playbook de Ansible para instalar y configurar `nftables` en los nodos.

## Requisitos Previos

1. **Ansible**: Asegúrate de tener Ansible instalado en tu máquina de control.
2. **Acceso a los nodos**: Los nodos deben ser accesibles mediante SSH y tener privilegios de superusuario.

## Uso

1. Configura el archivo de inventario `inventory/hosts.ini` con los detalles de tus nodos.
2. Ejecuta el playbook de Ansible para configurar `nftables`:

    ```bash
    sudo ansible-playbook -i inventory/hosts.ini playbooks/setup_nftables.yml
    ```

## Detalles de Configuración

El archivo `nftables.conf` define las reglas de firewall necesarias para:

- Permitir tráfico en puertos específicos (por ejemplo, 22, 6443, 8080, etc.).
- Configurar reglas NAT para el enrutamiento de tráfico entre subredes.
- Asegurar políticas predeterminadas para entrada, reenvío y salida.

## Comandos Útiles

- Aplicar las reglas de `nftables` manualmente:

    ```bash
    sudo nft -f /etc/sysconfig/nftables.conf
    ```

- Ver las reglas activas:

    ```bash
    sudo nft list ruleset
    ```

- Verificar el estado del servicio `nftables`:

    ```bash
    sudo systemctl status nftables
    ```

## Configuración de Rutas

### Load Balancers

#### load_balancer1 (10.17.3.12)

```bash
sudo ip route add 10.17.4.0/24 via 10.17.3.1 dev eth0   # Acceso a masters y workers
sudo ip route add 10.42.0.0/16 via 10.17.3.1 dev eth0   # Acceso a pods (Flannel)
sudo ip route add 10.17.5.0/24 via 10.17.3.1 dev eth0   # Acceso al VIP API Server
```

#### load_balancer2 (10.17.3.13)

```bash
sudo ip route add 10.17.4.0/24 via 10.17.3.1 dev eth0   # Acceso a masters y workers
sudo ip route add 10.42.0.0/16 via 10.17.3.1 dev eth0   # Acceso a pods (Flannel)
sudo ip route add 10.17.5.0/24 via 10.17.3.1 dev eth0   # Acceso al VIP API Server
```

### k8s-api-lb (10.17.5.20) — VM con HAProxy + Keepalived

#### Opción 1: Vía Gateway de Red

```bash
sudo ip route add 10.42.0.0/16 via 10.17.5.1 dev eth0
```

#### Opción 2: Vía un Nodo Master

```bash
sudo ip route add 10.42.0.0/16 via 10.17.4.21 dev eth0
```

### Comentarios

- Estas rutas son solo necesarias en los nodos fuera de la red `10.17.4.0/24` (load balancers y HAProxy).
- Los nodos master y worker (`10.17.4.21–27`) ya tienen rutas nativas hacia `10.42.0.0/16` a través de Flannel (`flannel.1` y `cni0`), no requieren rutas extra.
- En los `load_balancerX`, la interfaz es `eth0`, y el gateway es `10.17.3.1` (seguramente pfSense o tu router interno).

## Licencia

Este proyecto está licenciado bajo los términos especificados en el archivo `LICENSE`.



🎯 Objetivo
Permitir que los balanceadores externos (load_balancer1, load_balancer2) puedan enrutar tráfico hacia los pods de cada nodo mediante rutas estáticas por subred.







✅ Comandos para load_balancer1 (IP: 10.17.3.12)
bash
Copiar
Editar
# worker1
sudo ip route add 10.42.0.0/24 via 10.17.4.24 dev eth0 onlink

# master1
sudo ip route add 10.42.1.0/24 via 10.17.4.21 dev eth0 onlink

# master2
sudo ip route add 10.42.3.0/24 via 10.17.4.22 dev eth0 onlink

# master3
sudo ip route add 10.42.4.0/24 via 10.17.4.23 dev eth0 onlink

# worker3 (tú estás en este nodo)
sudo ip route add 10.42.5.0/24 via 10.17.4.26 dev eth0 onlink

# worker2
sudo ip route add 10.42.6.0/24 via 10.17.4.25 dev eth0 onlink
sudo ip route add 10.42.9.0/24 via 10.17.4.25 dev eth0 onlink
✅ Comandos para load_balancer2 (IP: 10.17.3.13)
Iguales a los anteriores:

bash
Copiar
Editar
# worker1
sudo ip route add 10.42.0.0/24 via 10.17.4.24 dev eth0 onlink

# master1
sudo ip route add 10.42.1.0/24 via 10.17.4.21 dev eth0 onlink

# master2
sudo ip route add 10.42.3.0/24 via 10.17.4.22 dev eth0 onlink

# master3
sudo ip route add 10.42.4.0/24 via 10.17.4.23 dev eth0 onlink

# worker3
sudo ip route add 10.42.5.0/24 via 10.17.4.26 dev eth0 onlink

# worker2
sudo ip route add 10.42.6.0/24 via 10.17.4.25 dev eth0 onlink
sudo ip route add 10.42.9.0/24 via 10.17.4.25 dev eth0 onlink
🧪 Verificación
Después de añadir todas las rutas en ambos balanceadores, prueba desde load_balancer1:

bash
Copiar
Editar
ping -c 4 10.42.9.206
Y también:

bash
Copiar
Editar
curl http://10.42.9.206:8080  # si Jenkins escucha ahí






