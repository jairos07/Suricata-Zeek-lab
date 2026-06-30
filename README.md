# Suricata + Zeek NDR Sensor Lab

Lab de detección de red (NDR) desplegado sobre Proxmox VE con una VM Ubuntu Server que ejecuta Suricata IDS y Zeek vía Docker, con logs centralizados en Grafana + Loki + Promtail.

---

## Arquitectura

```
Proxmox VE 9.1.7
└── VM 305 · ndr-sensor · Ubuntu 24.04.4 LTS
    ├── ens18 → 192.168.1.228/24   (gestión / SSH)
    ├── ens19 → 192.168.100.4/24   (red monitorizada)
    └── ens20 → sin IP / PROMISC   (captura de tráfico)
         ├── Suricata 8.0.5        (af-packet)
         ├── Zeek (Docker)         (network host)
         └── NDR Stack (/opt/ndr-stack)
              ├── Promtail          (recolección de logs)
              ├── Loki :3100        (almacenamiento)
              └── Grafana :3000     (visualización)
```

---

## Índice

1. [Creación de la VM en Proxmox](#1-creación-de-la-vm-en-proxmox)
2. [Instalación de Ubuntu Server 24.04](#2-instalación-de-ubuntu-server-2404)
3. [Configuración de red y modo promiscuo](#3-configuración-de-red-y-modo-promiscuo)
4. [Instalación y configuración de Suricata](#4-instalación-y-configuración-de-suricata)
5. [Despliegue de Zeek con Docker](#5-despliegue-de-zeek-con-docker)
6. [Stack de monitorización: Grafana + Loki + Promtail](#6-stack-de-monitorización-grafana--loki--promtail)

---

## 1. Creación de la VM en Proxmox

Se creó la VM con los siguientes parámetros desde el asistente de Proxmox VE 9.1.7:

| Parámetro     | Valor                          | Por qué                                                    |
|---------------|--------------------------------|------------------------------------------------------------|
| VM ID / Nombre| 305 / ndr-sensor               | Identifica el rol del sensor dentro del inventario del lab |
| ISO           | ubuntu-24.04.4-live-server-amd64 | LTS estable, soporte largo plazo, sin entorno gráfico    |
| CPU           | 4 núcleos · x86-64-v2-AES      | Suricata escala por núcleo; AES-NI acelera inspección TLS  |
| Memoria       | 4 GB                           | Suficiente para Suricata (~213 MB) + Docker + OS           |
| Disco         | 40 GB · qcow2 · VirtIO SCSI   | qcow2 permite snapshots; VirtIO maximiza rendimiento I/O   |
| NIC 1 (net0)  | vmbr0 → ens18                  | Red de gestión, acceso SSH                                 |
| NIC 2 (net1)  | vmbr1 → ens19                  | Red interna monitorizada, con IP estática                  |
| NIC 3 (net2)  | vmbr1 → ens20                  | Captura promiscua, sin IP, comparte bridge con ens19       |

La tercera NIC se agregó tras la creación de la VM desde **Hardware → Agregar → Dispositivo de red**. Se asignó al mismo bridge `vmbr1` que `ens19` pero sin IP, para separar el plano de datos (captura) del plano de control (comunicación del sensor).

---

## 2. Instalación de Ubuntu Server 24.04

Instalación estándar con las siguientes decisiones:

- **Almacenamiento**: disco completo con LVM — permite redimensionar volúmenes en el futuro sin reparticionado.
- **Hostname**: `ndr-sensor` — aparece en el prompt y en los logs del sistema para facilitar la correlación de eventos.
- **Usuario**: `administrador`
- **SSH**: OpenSSH instalado durante la instalación con autenticación por contraseña habilitada para acceso remoto inmediato.

---

## 3. Configuración de red y modo promiscuo

### Configuración estática con Netplan

Se editó `/etc/netplan/50-cloud-init.yaml` para asignar IP estática a `ens19` y dejar `ens20` sin configurar:

```yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: true
    ens19:
      dhcp4: false
      addresses:
        - 192.168.100.4/24
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses:
          - 192.168.100.1
```

`ens20` no aparece en Netplan a propósito: asignarle una IP haría que el kernel procesara el tráfico capturado y alteraría el comportamiento del sniffing.

```bash
sudo netplan apply
```

### Modo promiscuo en ens20

El modo promiscuo permite que la NIC acepte todos los paquetes del segmento, independientemente de la MAC de destino. Sin él, Suricata y Zeek solo verían tráfico dirigido a la propia VM.

```bash
sudo ip link set ens20 promisc on
sudo ip link set ens20 up
```

### Persistencia con systemd

Para que el modo promiscuo sobreviva a reinicios se creó una unidad de servicio en `/etc/systemd/system/promisc-ens20.service`:

```ini
[Unit]
Description=Set ens20 promiscuous mode
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ip link set ens20 promisc on
ExecStart=/sbin/ip link set ens20 up
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable promisc-ens20
sudo systemctl start promisc-ens20
```

---

## 4. Instalación y configuración de Suricata

### Instalación

Se usó el PPA oficial de OISF para obtener la versión más reciente del canal estable:

```bash
sudo add-apt-repository ppa:oisf/suricata-stable
sudo apt install suricata suricata-update
```

Versión instalada: **Suricata 8.0.5 RELEASE**

### Configuración en `/etc/suricata/suricata.yaml`

**HOME_NET** — define la red interna a proteger, fundamental para que las reglas distingan tráfico entrante de saliente y eviten falsos positivos:

```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.100.0/24]"
```

**af-packet** — modo de captura nativo de Linux que lee paquetes directamente del kernel, ofreciendo el máximo rendimiento sin pasar por la pila de red completa:

```yaml
af-packet:
  - interface: ens20
```

### Descarga de reglas

```bash
sudo suricata-update
```

Descarga el conjunto **Emerging Threats Open**, obteniendo **50.882 reglas** que cubren amenazas conocidas, exploits, malware y comportamientos anómalos de red.

### Verificación y arranque

```bash
# Test de configuración sin capturar tráfico
sudo suricata -T -c /etc/suricata/suricata.yaml -v

# Arrancar el servicio
sudo systemctl restart suricata
sudo systemctl status suricata
```

Suricata escribe los eventos en `/var/log/suricata/eve.json` en formato EVE-JSON, con tipos de evento `alert`, `flow`, `dns`, `http`, `tls` y `stats`.

---

## 5. Despliegue de Zeek con Docker

Se eligió Docker para aislar Zeek del sistema base y simplificar actualizaciones de versión.

### Instalación de Docker

```bash
sudo apt install docker.io docker-compose
sudo systemctl enable docker
sudo systemctl start docker
```

### Lanzar el contenedor Zeek

```bash
sudo docker run -d \
  --name zeek \
  --network host \
  --cap-add NET_RAW \
  --cap-add NET_ADMIN \
  -v /opt/zeek-logs:/logs \
  zeek/zeek:latest \
  zeek -i ens20 LogAscii::use_json=T Log::default_logdir=/logs
```

- `--network host` — necesario para que el contenedor acceda directamente a `ens20`.
- `--cap-add NET_RAW / NET_ADMIN` — permisos de captura de paquetes raw.
- `-v /opt/zeek-logs:/logs` — los logs se persisten en el host fuera del contenedor.
- `LogAscii::use_json=T` — salida en JSON para compatibilidad con Promtail/Loki.

Zeek genera logs por tipo de protocolo (`conn.log`, `dns.log`, `http.log`, etc.) en `/opt/zeek-logs/`.

---

## 6. Stack de monitorización: Grafana + Loki + Promtail

El stack se despliega con Docker Compose en `/opt/ndr-stack/`. Promtail recolecta los logs del host, los envía a Loki donde se indexan, y Grafana los consulta y visualiza.

### `docker-compose.yml`

```yaml
version: '3.8'

services:
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    restart: unless-stopped

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - /var/log/suricata:/var/log/suricata:ro
      - /opt/zeek-logs:/var/log/zeek:ro
      - /opt/ndr-stack/promtail-config.yaml:/etc/promtail/config.yaml:ro
    command: -config.file=/etc/promtail/config.yaml
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=<contraseña>
    restart: unless-stopped
```

### `promtail-config.yaml`

Promtail observa tres archivos de log y los etiqueta para poder filtrarlos en Grafana:

```yaml
clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: suricata
    static_configs:
      - targets: [localhost]
        labels:
          job: suricata
          __path__: /var/log/suricata/eve.json

  - job_name: zeek_conn
    static_configs:
      - targets: [localhost]
        labels:
          job: zeek
          log_type: conn
          __path__: /var/log/zeek/conn.log

  - job_name: zeek_dns
    static_configs:
      - targets: [localhost]
        labels:
          job: zeek
          log_type: dns
          __path__: /var/log/zeek/dns.log
```

### Arranque y verificación

```bash
cd /opt/ndr-stack
sudo docker-compose up -d
sudo docker-compose ps
# grafana   Up   0.0.0.0:3000->3000/tcp
# loki      Up   0.0.0.0:3100->3100/tcp
# promtail  Up
```

### Configuración en Grafana

1. Acceder a `http://192.168.100.4:3000`
2. **Connections → Add new connection → Loki**
3. URL: `http://loki:3100` (resolución por nombre de contenedor dentro de la red Docker)
4. Crear paneles con queries LogQL:
   - Logs de Suricata: `{job="suricata"}`
   - Logs de Zeek: `{job="zeek"}`

---

## Puertos del sistema

| Servicio | Puerto | Acceso                  |
|----------|--------|-------------------------|
| SSH      | 22     | 192.168.1.228 (ens18)   |
| Grafana  | 3000   | 192.168.100.4 (ens19)   |
| Loki     | 3100   | 192.168.100.4 (ens19)   |
