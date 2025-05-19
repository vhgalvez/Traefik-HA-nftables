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

## Licencia

Este proyecto está licenciado bajo los términos especificados en el archivo `LICENSE`.
