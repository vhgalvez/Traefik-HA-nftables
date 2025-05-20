# Guía de Comunicación de Red para Ingress Externo en Clúster K3s sobre KVM Bare Metal

## 📉 Objetivo

Permitir comunicación directa (ICMP y HTTP) desde los Load Balancer externos (Traefik en 10.17.3.12 y 10.17.3.13) hacia los pods del clúster Kubernetes (10.42.0.0/16), sin NAT, utilizando rutas estáticas, proxy ARP y parámetros adecuados del kernel.

## 🗺️ Topología de Red

```
+-------------------------------------------------+
| HAProxy + Keepalived (VIP API Server)           |
| IP real: 10.17.5.20                             |
| VIP API Server: 10.17.5.10:6443                 |
+-------------------------------------------------+
                      |
                      v
   +-------------------+-------------------+-------------------+
   |                   |                   |                   |
   v                   v                   v
+---------------+ +---------------+ +---------------+
| Master Node 1 | | Master Node 2 | | Master Node 3 |
|   10.17.4.21  | |   10.17.4.22  | |   10.17.4.23  |
+---------------+ +---------------+ +---------------+
                      |
                      v
        +-------------------------------+
        |     Kubernetes API 6443       |
        +-------------------------------+
                      |
                      v
+--------------------------+     +--------------------------+
| Load Balancer 1 Traefik |     | Load Balancer 2 Traefik |
| IP: 10.17.3.12           |     | IP: 10.17.3.13           |
+--------------------------+     +--------------------------+
                      |
                      v
          +---------------------------------------+
          |      Worker Nodes (10.17.4.24-27)     |
          +---------------------------------------+
```

## 📊 Entorno

| Componente            | IP / Rango      | Descripción                     |
| --------------------- | --------------- | ------------------------------- |
| HAProxy + Keepalived  | 10.17.5.20      | Balanceador API K8s             |
| VIP Kubernetes API    | 10.17.5.10:6443 | Punto de entrada para API       |
| Traefik LB1           | 10.17.3.12      | Ingress externo (Docker)        |
| Traefik LB2           | 10.17.3.13      | Ingress externo (Docker)        |
| Master Nodes          | 10.17.4.21-23   | etcd + k3s                      |
| Worker Nodes          | 10.17.4.24-27   | Ejecutan pods (CNI Flannel)     |
| Red de pods           | 10.42.0.0/16    | Asignada por Flannel VXLAN      |
| Pod destino de prueba | 10.42.9.206     | Ubicado en worker2 (10.17.4.25) |

---

## 🛠️ Paso a Paso: Configuración del Clúster

### 🔧 En todos los nodos (masters y workers)

Agregar a `/etc/sysctl.conf` o aplicar con `sysctl -w`:

```bash
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=2
net.ipv4.conf.default.rp_filter=2
net.ipv4.conf.eth0.proxy_arp=1
```

Aplicar sin reiniciar:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv4.conf.all.rp_filter=2
sudo sysctl -w net.ipv4.conf.default.rp_filter=2
sudo sysctl -w net.ipv4.conf.eth0.proxy_arp=1
```

### 🔧 En cada nodo (masters y workers)

Verificar ruta de retorno hacia la red 10.17.3.0/24:

```bash
ip route | grep 10.17.3.0
# Si no está:
sudo ip route add 10.17.3.0/24 via 10.17.4.1 dev eth0
```

### 🔧 En los Load Balancer (10.17.3.12 / 10.17.3.13)

Agregar rutas estáticas hacia las subredes de pods:

```bash
sudo ip route add 10.42.0.0/24 via 10.17.4.24 dev eth0 onlink
sudo ip route add 10.42.1.0/24 via 10.17.4.21 dev eth0 onlink
sudo ip route add 10.42.3.0/24 via 10.17.4.22 dev eth0 onlink
sudo ip route add 10.42.4.0/24 via 10.17.4.23 dev eth0 onlink
sudo ip route add 10.42.5.0/24 via 10.17.4.26 dev eth0 onlink
sudo ip route add 10.42.6.0/24 via 10.17.4.25 dev eth0 onlink
sudo ip route add 10.42.9.0/24 via 10.17.4.25 dev eth0 onlink
```

### ❌ Evitar NAT (masquerade)

Si se usa `ip-masq-agent`, configurar:

**Archivo:** `/etc/config/ip-masq-agent`

```json
{
  "nonMasqueradeCIDRs": ["10.17.0.0/16"]
}
```

En caso de usar `nftables`, **evitar**:

```nft
ip saddr 10.17.3.0/24 ip daddr 10.42.0.0/16 masquerade
```

---

## 🔍 Validación Paso a Paso

### Desde Load Balancer (10.17.3.13)

```bash
ping -c 4 10.17.4.25
ping -c 4 10.42.9.206
curl http://10.42.9.206:8080
```

### En Worker2 (10.17.4.25)

```bash
sudo tcpdump -i eth0 icmp and host 10.17.3.13
sudo tcpdump -i flannel.1 icmp
sudo tcpdump -i eth0 icmp and src 10.42.9.206
```

---

## 📊 Resultado Esperado

* Tráfico desde los Load Balancer hacia cualquier pod debe funcionar sin NAT.
* Las respuestas del pod deben conservar su IP original (`10.42.x.x`).
* Traefik puede realizar health-checks y balancear peticiones sin interferencias.

---

## 📊 Recomendación: Persistencia de rutas

* En Flatcar: usar `ignition` o `networkd` drop-ins.
* En Rocky Linux: agregar en `/etc/sysconfig/network-scripts/route-eth0`:

```bash
10.17.3.0/24 via 10.17.4.1 dev eth0
```

---

## 📊 Nodos donde aplicar ruta de retorno a 10.17.3.0/24

| Nodo    | IP         | Estado        |
| ------- | ---------- | ------------- |
| master1 | 10.17.4.21 | Requiere ruta |
| master2 | 10.17.4.22 | Requiere ruta |
| master3 | 10.17.4.23 | Requiere ruta |
| worker1 | 10.17.4.24 | Requiere ruta |
| worker2 | 10.17.4.25 | ✅ Ya aplicada |
| worker3 | 10.17.4.26 | Requiere ruta |
| worker4 | 10.17.4.27 | Requiere ruta |




core@worker1 ~ $ sudo iptables --version
iptables v1.8.8 (nf_tables)
core@worker1 ~ $ sudo nft add rule ip nat postrouting ip saddr 10.17.0.0/16 ip daddr 10.42.0.0/16 masquerade
Error: Could not process rule: No such file or directory
add rule ip nat postrouting ip saddr 10.17.0.0/16 ip daddr 10.42.0.0/16 masquerade
                ^^^^^^^^^^^
core@worker1 ~ $ sudo nft add table ip nat
core@worker1 ~ $ sudo nft add chain ip nat postrouting { type nat hook postrouting priority 100 \; }
core@worker1 ~ $ sudo nft add rule ip nat postrouting ip saddr 10.17.0.0/16 ip daddr 10.42.0.0/16 masquerade
core@worker1 ~ $ sudo nft list table ip nat
table ip nat {
