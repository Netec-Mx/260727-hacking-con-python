---LAB_START---
LAB_ID: 06-00-01
---MARKDOWN---
# Práctica 6 — Scripts de Scapy y Rutinas SSH Controladas

## 1. Metadatos

| Campo            | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 46 minutos                                 |
| **Complejidad**  | Alta (Hard)                                |
| **Nivel Bloom**  | Crear (Create)                             |
| **Módulo**       | 6 — Sockets, Scapy y Automatización SSH    |
| **Versión guía** | 1.0                                        |

---

## 2. Descripción General

En este laboratorio el estudiante construirá, desde cero, un mini-toolkit de reconocimiento activo compuesto por dos partes complementarias. La **Parte A** utiliza Scapy para realizar ARP scanning, SYN scanning y captura pasiva de tráfico en la red NAT interna. La **Parte B** emplea Paramiko para automatizar conexiones SSH a Metasploitable 2, ejecutar comandos de enumeración y transferir archivos de resultados al host local. Al finalizar, ambas partes se integran en un único script orquestador que descubre hosts, verifica el puerto SSH y se conecta automáticamente, aplicando los principios del modelo cliente/servidor estudiados en la Lección 6.1.

> ⚠️ **AVISO ÉTICO Y LEGAL:** Todo el tráfico generado en este laboratorio debe permanecer **estrictamente dentro de la red NAT interna** de VirtualBox/VMware. Antes de comenzar, asegúrate de contar con el formulario de autorización firmado proporcionado por el instructor. Nunca ejecutes estos scripts fuera del entorno de laboratorio.

---

## 3. Objetivos de Aprendizaje

- [ ] Construir un ARP scanner y un SYN scanner con Scapy para descubrir hosts y puertos activos en la red interna sin completar el handshake TCP.
- [ ] Implementar un sniffer pasivo con Scapy que capture y clasifique paquetes por protocolo durante un intervalo de tiempo controlado.
- [ ] Automatizar conexiones SSH con Paramiko usando autenticación por contraseña y por clave pública para ejecutar comandos remotos y transferir archivos.
- [ ] Integrar los resultados de Scapy y Paramiko en un flujo de trabajo unificado que pase de descubrimiento de hosts a ejecución remota automatizada.

---

## 4. Prerrequisitos

### Conocimiento previo

- Haber completado **Lab 04-00-01** (conceptos de puertos y servicios).
- Comprender el modelo TCP/IP y el handshake de tres vías (SYN → SYN-ACK → ACK) según la Lección 6.1.
- Familiaridad con el ciclo de vida de un socket: `socket()` → `bind()/connect()` → `send()/recv()` → `close()`.
- Conocer la diferencia entre TCP (`SOCK_STREAM`) y UDP (`SOCK_DGRAM`) y sus usos en hacking ético.

### Acceso y entorno

- Kali Linux 2024.1+ como máquina atacante (host o VM).
- Metasploitable 2 activa con SSH habilitado en la red NAT interna.
- Ambas VMs conectadas a la **misma red NAT interna** (sin acceso a Internet durante la práctica).
- Privilegios `root` o `sudo` disponibles en Kali (requeridos por Scapy para raw sockets).
- Tarjeta de red de la VM Kali con **modo promiscuo habilitado**.
- Snapshot de Metasploitable 2 creado antes de iniciar (ver Sección 5).

---

## 5. Entorno de Laboratorio

### 5.1 Hardware recomendado

| Componente        | Mínimo                              | Recomendado                     |
|-------------------|-------------------------------------|---------------------------------|
| RAM               | 8 GB (4 GB por VM)                  | 16 GB                           |
| CPU               | 4 núcleos, 64 bits, VT-x/AMD-V      | 6–8 núcleos                     |
| Disco             | 60 GB libres                        | 100 GB SSD                      |
| Red               | NIC compatible con modo promiscuo   | NIC Gigabit                     |
| Resolución        | 1280×768                            | 1920×1080 (trabajo multi-terminal)|

### 5.2 Software requerido

| Herramienta        | Versión mínima | Instalación en Kali                    |
|--------------------|----------------|----------------------------------------|
| Python             | 3.10+          | Preinstalado                           |
| Scapy              | 2.5+           | `pip install scapy`                    |
| Paramiko           | 3.3+           | `pip install paramiko`                 |
| python-whois       | 0.8+           | `pip install python-whois`             |
| Metasploitable 2   | —              | VM importada en VirtualBox/VMware      |

### 5.3 Configuración de red NAT interna

Ambas VMs deben estar en la misma red interna aislada. En VirtualBox:

```bash
# En la configuración de red de CADA VM → Adaptador → Conectado a: "Red interna"
# Nombre de red: intnet_lab6   (usar el mismo nombre en ambas VMs)
```

### 5.4 Preparación del entorno (ejecutar antes de comenzar)

**Paso 0-A — Crear snapshot de Metasploitable 2 (OBLIGATORIO):**

```bash
# Desde el host, con la VM apagada o en estado guardado:
# VirtualBox → seleccionar "Metasploitable2" → Máquina → Tomar Snapshot
# Nombre sugerido: "Lab06_inicio_limpio"
# Alternativamente, desde línea de comandos:
VBoxManage snapshot "Metasploitable2" take "Lab06_inicio_limpio" --description "Estado limpio antes de Lab 06-00-01"
```

**Paso 0-B — Verificar conectividad en la red interna:**

```bash
# En Kali Linux — identificar la interfaz de red interna
ip addr show
# Ejemplo de salida esperada: eth1 con IP en rango 192.168.56.0/24 o 10.0.2.0/24

# Anotar la IP de Kali y el rango de red para usarlos en los scripts
export KALI_IP="192.168.56.101"        # Ajustar según tu entorno
export TARGET_NETWORK="192.168.56.0/24" # Ajustar según tu entorno
export METASPLOITABLE_IP="192.168.56.102" # Ajustar según tu entorno
export IFACE="eth1"                    # Interfaz de red interna
```

**Paso 0-C — Habilitar modo promiscuo en la interfaz:**

```bash
sudo ip link set $IFACE promisc on
ip link show $IFACE | grep -i promisc   # Verificar: debe aparecer "PROMISC"
```

**Paso 0-D — Crear directorio de trabajo y entorno virtual:**

```bash
mkdir -p ~/lab06/{parte_a,parte_b,integrado,resultados}
cd ~/lab06
python3 -m venv venv
source venv/bin/activate
pip install scapy paramiko
```

**Paso 0-E — Verificar que Metasploitable 2 tiene SSH activo:**

```bash
# Desde Kali, verificar puerto 22 usando el concepto de connect_ex() de la Lección 6.1
python3 -c "
import socket
host = '$METASPLOITABLE_IP'
puerto = 22
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.settimeout(2.0)
    resultado = s.connect_ex((host, puerto))
    print('SSH ABIERTO' if resultado == 0 else f'SSH NO DISPONIBLE (código: {resultado})')
"
```

---

## 6. Procedimiento Paso a Paso

### ─── PARTE A: Scapy — Análisis y Construcción de Paquetes ───

---

### Paso 1 — ARP Scanner: Descubrimiento de Hosts Activos

**Objetivo:** Construir un scanner ARP que identifique todos los hosts activos en la red NAT interna enviando solicitudes ARP broadcast y analizando las respuestas.

#### Instrucciones

1. Crear el archivo `~/lab06/parte_a/arp_scanner.py`:

```python
#!/usr/bin/env python3
"""
arp_scanner.py — ARP Scanner con Scapy
Lab 06-00-01 | Parte A — Paso 1
ADVERTENCIA: Ejecutar ÚNICAMENTE en la red interna de laboratorio con autorización.
"""

import sys
from scapy.all import ARP, Ether, srp, conf

# Suprimir la salida verbose de Scapy durante el envío
conf.verb = 0


def arp_scan(red: str, iface: str, timeout: float = 2.0) -> list[dict]:
    """
    Envía solicitudes ARP broadcast al rango de red indicado.
    Retorna lista de dicts con 'ip' y 'mac' de cada host que responde.

    Args:
        red:     Rango CIDR objetivo, ej. "192.168.56.0/24"
        iface:   Interfaz de red a usar, ej. "eth1"
        timeout: Tiempo máximo de espera para respuestas (segundos)

    Returns:
        Lista de diccionarios [{"ip": "...", "mac": "..."}]
    """
    print(f"[*] Iniciando ARP scan sobre {red} (interfaz: {iface})")
    print("[*] Construyendo paquete ARP broadcast...")

    # Capa Ethernet: destino ff:ff:ff:ff:ff:ff = broadcast
    capa_ether = Ether(dst="ff:ff:ff:ff:ff:ff")

    # Capa ARP: pdst = IP destino (rango CIDR)
    # op=1 (who-has) es el valor por defecto; lo dejamos explícito para claridad
    capa_arp = ARP(pdst=red, op=1)

    # Combinamos las capas con el operador / de Scapy
    paquete = capa_ether / capa_arp

    print(f"[*] Enviando {paquete.summary()} ...")

    # srp() = send/receive en capa 2; retorna (respondidos, no_respondidos)
    respondidos, _ = srp(paquete, iface=iface, timeout=timeout, retry=1)

    hosts_activos = []
    for enviado, recibido in respondidos:
        hosts_activos.append({
            "ip":  recibido[ARP].psrc,   # IP de quien respondió
            "mac": recibido[Ether].src   # MAC de quien respondió
        })

    return hosts_activos


def mostrar_resultados(hosts: list[dict]) -> None:
    """Imprime la tabla de hosts descubiertos."""
    print(f"\n{'─'*45}")
    print(f"  {'IP':<20} {'MAC':<20}")
    print(f"{'─'*45}")
    for host in hosts:
        print(f"  {host['ip']:<20} {host['mac']:<20}")
    print(f"{'─'*45}")
    print(f"[+] Total de hosts activos: {len(hosts)}")


def guardar_resultados(hosts: list[dict], archivo: str) -> None:
    """Guarda los resultados en un archivo de texto."""
    with open(archivo, "w") as f:
        f.write("ip,mac\n")
        for host in hosts:
            f.write(f"{host['ip']},{host['mac']}\n")
    print(f"[+] Resultados guardados en: {archivo}")


if __name__ == "__main__":
    # Parámetros — ajustar según el entorno del laboratorio
    RED_OBJETIVO = sys.argv[1] if len(sys.argv) > 1 else "192.168.56.0/24"
    INTERFAZ     = sys.argv[2] if len(sys.argv) > 2 else "eth1"
    ARCHIVO_OUT  = "/root/lab06/resultados/hosts_arp.csv"

    hosts = arp_scan(RED_OBJETIVO, INTERFAZ)
    mostrar_resultados(hosts)
    guardar_resultados(hosts, ARCHIVO_OUT)
```

2. Ejecutar el script con privilegios root (requerido por Scapy para raw sockets):

```bash
cd ~/lab06
source venv/bin/activate
sudo python3 parte_a/arp_scanner.py $TARGET_NETWORK $IFACE
```

3. Verificar que el archivo de resultados se generó correctamente:

```bash
cat ~/lab06/resultados/hosts_arp.csv
```

#### Salida esperada

```
[*] Iniciando ARP scan sobre 192.168.56.0/24 (interfaz: eth1)
[*] Construyendo paquete ARP broadcast...
[*] Enviando Ether / ARP who has 192.168.56.0/24 ...
─────────────────────────────────────────────
  IP                   MAC
─────────────────────────────────────────────
  192.168.56.1         0a:00:27:00:00:00
  192.168.56.101       08:00:27:ab:cd:ef
  192.168.56.102       08:00:27:12:34:56
─────────────────────────────────────────────
[+] Total de hosts activos: 3
[+] Resultados guardados en: /root/lab06/resultados/hosts_arp.csv
```

#### Verificación

```bash
# Comparar con arping nativo para validar resultados
sudo arping -c 3 -I $IFACE $METASPLOITABLE_IP

# Verificar que el CSV contiene la IP de Metasploitable 2
grep "$METASPLOITABLE_IP" ~/lab06/resultados/hosts_arp.csv && echo "✓ Metasploitable 2 detectado"
```

---

### Paso 2 — SYN Scanner: Verificación de Puertos sin Completar el Handshake

**Objetivo:** Implementar un SYN scanner que envíe paquetes TCP SYN y analice las respuestas (SYN-ACK = abierto, RST = cerrado) sin completar el handshake de tres vías, aplicando los conceptos de la Lección 6.1.

#### Instrucciones

1. Crear el archivo `~/lab06/parte_a/syn_scanner.py`:

```python
#!/usr/bin/env python3
"""
syn_scanner.py — SYN Scanner (Half-Open) con Scapy
Lab 06-00-01 | Parte A — Paso 2
ADVERTENCIA: Ejecutar ÚNICAMENTE en la red interna de laboratorio con autorización.

Fundamento (Lección 6.1):
  El handshake TCP de tres vías es: SYN → SYN-ACK → ACK
  Un SYN scan envía solo el primer paso (SYN) y observa la respuesta:
    - SYN-ACK recibido  → puerto ABIERTO (enviamos RST para no completar conexión)
    - RST recibido      → puerto CERRADO
    - Sin respuesta     → puerto FILTRADO (firewall)
"""

import random
import sys
from scapy.all import IP, TCP, sr1, conf, send

conf.verb = 0  # Silenciar output de Scapy


def syn_scan(host: str, puertos: list[int], timeout: float = 1.0) -> dict[int, str]:
    """
    Realiza un SYN scan sobre los puertos indicados del host objetivo.

    Args:
        host:    IP del host objetivo
        puertos: Lista de puertos a escanear
        timeout: Tiempo máximo de espera por respuesta (segundos)

    Returns:
        Diccionario {puerto: estado} donde estado es "ABIERTO", "CERRADO" o "FILTRADO"
    """
    print(f"\n[*] SYN Scan sobre {host} — {len(puertos)} puertos")
    print(f"[*] Timeout por puerto: {timeout}s\n")

    resultados = {}

    for puerto in puertos:
        # Puerto origen aleatorio (evita conflictos con el stack TCP del OS)
        puerto_origen = random.randint(1024, 65535)

        # Construir paquete: IP / TCP con flag SYN
        # flags="S" = SYN; equivale a flags=0x02
        paquete_syn = IP(dst=host) / TCP(
            sport=puerto_origen,
            dport=puerto,
            flags="S",
            seq=random.randint(0, 2**32 - 1)  # Número de secuencia aleatorio
        )

        # sr1() = send/receive 1 paquete en capa 3
        respuesta = sr1(paquete_syn, timeout=timeout, verbose=0)

        if respuesta is None:
            # Sin respuesta → puerto filtrado por firewall
            estado = "FILTRADO"

        elif respuesta.haslayer(TCP):
            flags_resp = respuesta[TCP].flags

            if flags_resp == "SA":  # SYN-ACK (0x12) → puerto abierto
                estado = "ABIERTO"
                # Enviamos RST para evitar completar el handshake y no crear
                # una conexión real en el host objetivo
                rst = IP(dst=host) / TCP(
                    sport=puerto_origen,
                    dport=puerto,
                    flags="R",
                    seq=respuesta[TCP].ack
                )
                send(rst, verbose=0)

            elif flags_resp == "RA" or flags_resp == "R":  # RST → cerrado
                estado = "CERRADO"
            else:
                estado = f"DESCONOCIDO (flags={flags_resp})"
        else:
            estado = "FILTRADO"

        resultados[puerto] = estado

        # Solo mostrar puertos abiertos y filtrados para reducir ruido
        if estado != "CERRADO":
            print(f"  Puerto {puerto:5d}/TCP  →  {estado}")

    return resultados


def resumen_scan(host: str, resultados: dict[int, str]) -> None:
    """Imprime un resumen del escaneo."""
    abiertos  = [p for p, e in resultados.items() if e == "ABIERTO"]
    cerrados  = [p for p, e in resultados.items() if e == "CERRADO"]
    filtrados = [p for p, e in resultados.items() if e == "FILTRADO"]

    print(f"\n{'═'*50}")
    print(f"  Resumen SYN Scan — {host}")
    print(f"{'═'*50}")
    print(f"  Puertos abiertos  : {len(abiertos)}")
    print(f"  Puertos cerrados  : {len(cerrados)}")
    print(f"  Puertos filtrados : {len(filtrados)}")
    if abiertos:
        print(f"\n  Puertos ABIERTOS  : {', '.join(map(str, sorted(abiertos)))}")
    print(f"{'═'*50}")


if __name__ == "__main__":
    HOST_OBJETIVO = sys.argv[1] if len(sys.argv) > 1 else "192.168.56.102"

    # Puertos comunes en Metasploitable 2
    PUERTOS_OBJETIVO = [21, 22, 23, 25, 80, 139, 445, 3306, 5432, 8180]

    resultados = syn_scan(HOST_OBJETIVO, PUERTOS_OBJETIVO)
    resumen_scan(HOST_OBJETIVO, resultados)
```

2. Ejecutar el SYN scanner:

```bash
sudo python3 parte_a/syn_scanner.py $METASPLOITABLE_IP
```

3. Comparar resultados con Nmap para validación:

```bash
# Comparación con herramienta nativa (solo para validación en el laboratorio)
sudo nmap -sS -p 21,22,23,25,80,139,445,3306,5432,8180 $METASPLOITABLE_IP
```

#### Salida esperada

```
[*] SYN Scan sobre 192.168.56.102 — 10 puertos
[*] Timeout por puerto: 1.0s

  Puerto    21/TCP  →  ABIERTO
  Puerto    22/TCP  →  ABIERTO
  Puerto    23/TCP  →  ABIERTO
  Puerto    25/TCP  →  ABIERTO
  Puerto    80/TCP  →  ABIERTO
  Puerto   139/TCP  →  ABIERTO
  Puerto   445/TCP  →  ABIERTO
  Puerto  3306/TCP  →  ABIERTO
  Puerto  5432/TCP  →  ABIERTO
  Puerto  8180/TCP  →  ABIERTO

══════════════════════════════════════════════════
  Resumen SYN Scan — 192.168.56.102
══════════════════════════════════════════════════
  Puertos abiertos  : 10
  Puertos cerrados  : 0
  Puertos filtrados : 0

  Puertos ABIERTOS  : 21, 22, 23, 25, 80, 139, 445, 3306, 5432, 8180
══════════════════════════════════════════════════
```

#### Verificación

```bash
# Verificar que el RST se envió correctamente (no debe haber conexiones ESTABLISHED)
# En Kali, durante el scan:
ss -tn | grep $METASPLOITABLE_IP
# Resultado esperado: ninguna línea (sin conexiones completadas)
```

---

### Paso 3 — Sniffer Pasivo: Captura y Clasificación de Tráfico

**Objetivo:** Implementar un sniffer pasivo con Scapy que capture paquetes durante 30 segundos y los clasifique por protocolo, mostrando IPs origen/destino y puertos en tiempo real.

#### Instrucciones

1. Crear el archivo `~/lab06/parte_a/sniffer_pasivo.py`:

```python
#!/usr/bin/env python3
"""
sniffer_pasivo.py — Sniffer Pasivo con Scapy
Lab 06-00-01 | Parte A — Paso 3
ADVERTENCIA: Captura ÚNICAMENTE tráfico de la red interna de laboratorio.
"""

import sys
import time
from collections import defaultdict
from scapy.all import sniff, IP, TCP, UDP, ICMP, ARP, Ether, conf

conf.verb = 0

# Contador global de paquetes por protocolo
estadisticas: dict[str, int] = defaultdict(int)
paquetes_capturados: list[dict] = []


def clasificar_paquete(pkt) -> None:
    """
    Callback invocado por sniff() para cada paquete capturado.
    Clasifica el paquete por protocolo y extrae información relevante.
    """
    timestamp = time.strftime("%H:%M:%S")
    info = {"tiempo": timestamp, "protocolo": "DESCONOCIDO", "detalle": ""}

    if pkt.haslayer(ARP):
        estadisticas["ARP"] += 1
        info["protocolo"] = "ARP"
        op = "who-has" if pkt[ARP].op == 1 else "is-at"
        info["detalle"] = f"{pkt[ARP].psrc} → {pkt[ARP].pdst} ({op})"

    elif pkt.haslayer(IP):
        src_ip = pkt[IP].src
        dst_ip = pkt[IP].dst

        if pkt.haslayer(TCP):
            estadisticas["TCP"] += 1
            info["protocolo"] = "TCP"
            sport = pkt[TCP].sport
            dport = pkt[TCP].dport
            flags = pkt[TCP].flags
            info["detalle"] = f"{src_ip}:{sport} → {dst_ip}:{dport} [flags={flags}]"

        elif pkt.haslayer(UDP):
            estadisticas["UDP"] += 1
            info["protocolo"] = "UDP"
            sport = pkt[UDP].sport
            dport = pkt[UDP].dport
            info["detalle"] = f"{src_ip}:{sport} → {dst_ip}:{dport}"

        elif pkt.haslayer(ICMP):
            estadisticas["ICMP"] += 1
            info["protocolo"] = "ICMP"
            tipo = pkt[ICMP].type
            info["detalle"] = f"{src_ip} → {dst_ip} (type={tipo})"

        else:
            estadisticas["IP_OTRO"] += 1
            info["protocolo"] = "IP_OTRO"
            info["detalle"] = f"{src_ip} → {dst_ip}"
    else:
        estadisticas["OTRO"] += 1
        info["protocolo"] = "OTRO"

    paquetes_capturados.append(info)

    # Mostrar en tiempo real (solo TCP, UDP, ICMP y ARP para no saturar)
    if info["protocolo"] in ("TCP", "UDP", "ICMP", "ARP"):
        print(f"  [{info['tiempo']}] {info['protocolo']:<8} {info['detalle']}")


def mostrar_estadisticas() -> None:
    """Muestra el resumen estadístico al finalizar la captura."""
    total = sum(estadisticas.values())
    print(f"\n{'═'*55}")
    print(f"  RESUMEN DE CAPTURA — {total} paquetes totales")
    print(f"{'═'*55}")
    for proto, cantidad in sorted(estadisticas.items(), key=lambda x: -x[1]):
        porcentaje = (cantidad / total * 100) if total > 0 else 0
        barra = "█" * int(porcentaje / 5)
        print(f"  {proto:<12} {cantidad:>5} paquetes  {porcentaje:5.1f}%  {barra}")
    print(f"{'═'*55}")


def guardar_captura(archivo: str) -> None:
    """Guarda los paquetes clasificados en un archivo de texto."""
    with open(archivo, "w") as f:
        f.write("tiempo,protocolo,detalle\n")
        for p in paquetes_capturados:
            f.write(f"{p['tiempo']},{p['protocolo']},{p['detalle']}\n")
    print(f"[+] Captura guardada en: {archivo}")


if __name__ == "__main__":
    IFACE    = sys.argv[1] if len(sys.argv) > 1 else "eth1"
    DURACION = int(sys.argv[2]) if len(sys.argv) > 2 else 30
    ARCHIVO  = "/root/lab06/resultados/captura_pasiva.csv"

    print(f"[*] Iniciando sniffer pasivo en interfaz '{IFACE}' durante {DURACION} segundos...")
    print(f"[*] Genera tráfico desde otra terminal para capturar paquetes.")
    print(f"[*] Sugerencia: ejecuta 'ping {sys.argv[3] if len(sys.argv) > 3 else \"192.168.56.102\"}' en otra terminal.\n")

    # sniff() con timeout para limitar la duración de la captura
    # prn=clasificar_paquete → callback por cada paquete
    # store=False → no almacenar en memoria interna de Scapy (ahorramos RAM)
    sniff(
        iface=IFACE,
        prn=clasificar_paquete,
        timeout=DURACION,
        store=False
    )

    mostrar_estadisticas()
    guardar_captura(ARCHIVO)
```

2. Abrir **dos terminales** en Kali Linux. En la **Terminal 1**, ejecutar el sniffer:

```bash
sudo python3 parte_a/sniffer_pasivo.py $IFACE 30 $METASPLOITABLE_IP
```

3. En la **Terminal 2**, generar tráfico de prueba durante los 30 segundos:

```bash
# Generar tráfico ICMP
ping -c 5 $METASPLOITABLE_IP

# Generar tráfico TCP (HTTP)
curl -s http://$METASPLOITABLE_IP/ > /dev/null

# Generar tráfico ARP
sudo arping -c 3 -I $IFACE $METASPLOITABLE_IP
```

#### Salida esperada (Terminal 1)

```
[*] Iniciando sniffer pasivo en interfaz 'eth1' durante 30 segundos...
[*] Genera tráfico desde otra terminal para capturar paquetes.

  [14:22:01] ARP      192.168.56.101 → 192.168.56.102 (who-has)
  [14:22:01] ARP      192.168.56.102 → 192.168.56.101 (is-at)
  [14:22:01] ICMP     192.168.56.101 → 192.168.56.102 (type=8)
  [14:22:01] ICMP     192.168.56.102 → 192.168.56.101 (type=0)
  [14:22:03] TCP      192.168.56.101:54321 → 192.168.56.102:80 [flags=S]
  [14:22:03] TCP      192.168.56.102:80 → 192.168.56.101:54321 [flags=SA]
  ...

═══════════════════════════════════════════════════════
  RESUMEN DE CAPTURA — 47 paquetes totales
═══════════════════════════════════════════════════════
  TCP          28 paquetes   59.6%  ████████████
  ICMP         10 paquetes   21.3%  ████
  ARP           6 paquetes   12.8%  ██
  UDP           3 paquetes    6.4%  █
═══════════════════════════════════════════════════════
[+] Captura guardada en: /root/lab06/resultados/captura_pasiva.csv
```

#### Verificación

```bash
# Verificar que el CSV de captura contiene registros de múltiples protocolos
awk -F',' '{print $2}' ~/lab06/resultados/captura_pasiva.csv | sort | uniq -c
```

---

### ─── PARTE B: Paramiko — Automatización SSH ───

---

### Paso 4 — Conexión SSH con Autenticación por Contraseña

**Objetivo:** Establecer una conexión SSH a Metasploitable 2 con Paramiko usando autenticación por contraseña y ejecutar comandos de enumeración básica.

#### Instrucciones

1. Crear el archivo `~/lab06/parte_b/ssh_password.py`:

```python
#!/usr/bin/env python3
"""
ssh_password.py — Conexión SSH con Paramiko (autenticación por contraseña)
Lab 06-00-01 | Parte B — Paso 4
ADVERTENCIA: Usar ÚNICAMENTE sobre Metasploitable 2 en red interna de laboratorio.
"""

import sys
import paramiko
from pathlib import Path


def conectar_ssh_password(
    host: str,
    puerto: int,
    usuario: str,
    password: str,
    timeout: float = 10.0
) -> paramiko.SSHClient | None:
    """
    Establece una conexión SSH usando autenticación por contraseña.

    Args:
        host:     IP o hostname del servidor SSH
        puerto:   Puerto SSH (normalmente 22)
        usuario:  Nombre de usuario
        password: Contraseña del usuario
        timeout:  Tiempo máximo de conexión

    Returns:
        Objeto SSHClient conectado, o None si la conexión falla
    """
    cliente = paramiko.SSHClient()

    # AutoAddPolicy acepta automáticamente la clave del host desconocido.
    # En producción real se usaría RejectPolicy + known_hosts verificado.
    # En laboratorio controlado es aceptable para simplificar la práctica.
    cliente.set_missing_host_key_policy(paramiko.AutoAddPolicy())

    try:
        print(f"[*] Conectando a {host}:{puerto} como '{usuario}'...")
        cliente.connect(
            hostname=host,
            port=puerto,
            username=usuario,
            password=password,
            timeout=timeout,
            look_for_keys=False,   # No buscar claves en ~/.ssh/
            allow_agent=False      # No usar SSH agent
        )
        print(f"[+] Conexión SSH establecida con {host}")
        return cliente

    except paramiko.AuthenticationException:
        print(f"[!] Error de autenticación: credenciales incorrectas para '{usuario}'")
        return None
    except paramiko.SSHException as e:
        print(f"[!] Error SSH: {e}")
        return None
    except Exception as e:
        print(f"[!] Error de conexión: {e}")
        return None


def ejecutar_comando(cliente: paramiko.SSHClient, comando: str) -> tuple[str, str]:
    """
    Ejecuta un comando en el servidor SSH y retorna stdout y stderr.

    Args:
        cliente:  SSHClient conectado
        comando:  Comando a ejecutar

    Returns:
        Tupla (stdout, stderr)
    """
    stdin, stdout, stderr = cliente.exec_command(comando)
    salida = stdout.read().decode("utf-8", errors="replace").strip()
    error  = stderr.read().decode("utf-8", errors="replace").strip()
    return salida, error


def enumeracion_basica(cliente: paramiko.SSHClient) -> dict[str, str]:
    """
    Ejecuta comandos de enumeración básica y retorna los resultados.
    """
    comandos = {
        "sistema_operativo": "uname -a",
        "usuario_actual":    "id",
        "interfaces_red":    "ifconfig 2>/dev/null || ip addr show",
        "usuarios_sistema":  "cat /etc/passwd | cut -d: -f1,3,4,7 | head -20",
        "procesos_activos":  "ps aux --no-headers | head -15",
        "puertos_escucha":   "netstat -tlnp 2>/dev/null | head -20",
        "espacio_disco":     "df -h",
    }

    resultados = {}
    print(f"\n{'─'*50}")
    print("  ENUMERACIÓN BÁSICA DEL SISTEMA")
    print(f"{'─'*50}")

    for nombre, cmd in comandos.items():
        salida, error = ejecutar_comando(cliente, cmd)
        resultados[nombre] = salida if salida else error
        print(f"\n[+] {nombre.upper().replace('_', ' ')}:")
        print(f"    Comando: {cmd}")
        # Mostrar solo las primeras 3 líneas para no saturar la terminal
        lineas = salida.split("\n")[:3]
        for linea in lineas:
            print(f"    {linea}")
        if len(salida.split("\n")) > 3:
            print(f"    ... ({len(salida.split(chr(10)))} líneas totales)")

    return resultados


def guardar_enumeracion(resultados: dict[str, str], archivo: str) -> None:
    """Guarda los resultados de enumeración en un archivo."""
    Path(archivo).parent.mkdir(parents=True, exist_ok=True)
    with open(archivo, "w") as f:
        for clave, valor in resultados.items():
            f.write(f"\n{'='*50}\n")
            f.write(f"[{clave.upper()}]\n")
            f.write(f"{'='*50}\n")
            f.write(valor + "\n")
    print(f"\n[+] Enumeración guardada en: {archivo}")


if __name__ == "__main__":
    HOST     = sys.argv[1] if len(sys.argv) > 1 else "192.168.56.102"
    PUERTO   = int(sys.argv[2]) if len(sys.argv) > 2 else 22
    USUARIO  = "msfadmin"   # Credenciales por defecto de Metasploitable 2
    PASSWORD = "msfadmin"
    ARCHIVO  = "/root/lab06/resultados/enumeracion_ssh.txt"

    cliente = conectar_ssh_password(HOST, PUERTO, USUARIO, PASSWORD)

    if cliente:
        resultados = enumeracion_basica(cliente)
        guardar_enumeracion(resultados, ARCHIVO)
        cliente.close()
        print("[*] Conexión SSH cerrada correctamente.")
    else:
        print("[!] No se pudo establecer la conexión SSH.")
        sys.exit(1)
```

2. Ejecutar el script:

```bash
python3 parte_b/ssh_password.py $METASPLOITABLE_IP
```

> **Nota:** Este script **no** requiere `sudo` ya que Paramiko opera sobre sockets TCP estándar (capa de transporte), no sobre raw sockets.

#### Salida esperada

```
[*] Conectando a 192.168.56.102:22 como 'msfadmin'...
[+] Conexión SSH establecida con 192.168.56.102

──────────────────────────────────────────────────
  ENUMERACIÓN BÁSICA DEL SISTEMA
──────────────────────────────────────────────────

[+] SISTEMA OPERATIVO:
    Comando: uname -a
    Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux

[+] USUARIO ACTUAL:
    Comando: id
    uid=1000(msfadmin) gid=1000(msfadmin) groups=4(adm),20(dialout),...

[+] INTERFACES RED:
    Comando: ifconfig 2>/dev/null || ip addr show
    eth0      Link encap:Ethernet  HWaddr 08:00:27:12:34:56
    inet addr:192.168.56.102  Bcast:192.168.56.255  Mask:255.255.255.0
    ...

[+] Enumeración guardada en: /root/lab06/resultados/enumeracion_ssh.txt
[*] Conexión SSH cerrada correctamente.
```

#### Verificación

```bash
# Verificar que el archivo de enumeración contiene datos reales
grep -i "metasploitable\|linux" ~/lab06/resultados/enumeracion_ssh.txt
wc -l ~/lab06/resultados/enumeracion_ssh.txt
```

---

### Paso 5 — Autenticación SSH por Clave Pública y Transferencia de Archivos

**Objetivo:** Configurar autenticación SSH por clave pública y transferir el archivo de enumeración al host local usando SFTP a través de Paramiko.

#### Instrucciones

1. Generar un par de claves SSH en Kali (si no existe):

```bash
# Generar clave RSA de 4096 bits para el laboratorio
ssh-keygen -t rsa -b 4096 -f ~/.ssh/lab06_key -N "" -C "lab06-kali"

# Verificar que se generaron ambos archivos
ls -la ~/.ssh/lab06_key*
# Esperado: lab06_key (privada) y lab06_key.pub (pública)
```

2. Copiar la clave pública a Metasploitable 2:

```bash
# Método 1: ssh-copy-id (si está disponible)
ssh-copy-id -i ~/.ssh/lab06_key.pub -p 22 msfadmin@$METASPLOITABLE_IP

# Método 2: Manual (si ssh-copy-id no funciona con versión antigua de SSH)
cat ~/.ssh/lab06_key.pub | ssh msfadmin@$METASPLOITABLE_IP \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

3. Crear el archivo `~/lab06/parte_b/ssh_clave_sftp.py`:

```python
#!/usr/bin/env python3
"""
ssh_clave_sftp.py — SSH con clave pública + transferencia SFTP
Lab 06-00-01 | Parte B — Paso 5
"""

import sys
import os
import paramiko
from pathlib import Path


def conectar_ssh_clave(
    host: str,
    puerto: int,
    usuario: str,
    ruta_clave: str,
    timeout: float = 10.0
) -> paramiko.SSHClient | None:
    """
    Establece conexión SSH usando autenticación por clave privada RSA.
    """
    cliente = paramiko.SSHClient()
    cliente.set_missing_host_key_policy(paramiko.AutoAddPolicy())

    try:
        # Cargar la clave privada desde el archivo
        clave_privada = paramiko.RSAKey.from_private_key_file(ruta_clave)
        print(f"[*] Clave privada cargada: {ruta_clave}")

        print(f"[*] Conectando a {host}:{puerto} con clave pública...")
        cliente.connect(
            hostname=host,
            port=puerto,
            username=usuario,
            pkey=clave_privada,
            timeout=timeout,
            look_for_keys=False,
            allow_agent=False
        )
        print(f"[+] Autenticación por clave pública exitosa en {host}")
        return cliente

    except paramiko.AuthenticationException:
        print("[!] Autenticación fallida: clave pública no autorizada en el servidor")
        return None
    except FileNotFoundError:
        print(f"[!] No se encontró el archivo de clave: {ruta_clave}")
        return None
    except Exception as e:
        print(f"[!] Error: {e}")
        return None


def transferir_archivo_sftp(
    cliente: paramiko.SSHClient,
    ruta_local: str,
    ruta_remota: str,
    direccion: str = "subir"
) -> bool:
    """
    Transfiere un archivo usando SFTP.

    Args:
        cliente:      SSHClient conectado
        ruta_local:   Ruta del archivo en el sistema local
        ruta_remota:  Ruta del archivo en el servidor remoto
        direccion:    "subir" (local→remoto) o "descargar" (remoto→local)

    Returns:
        True si la transferencia fue exitosa
    """
    try:
        sftp = cliente.open_sftp()

        if direccion == "subir":
            sftp.put(ruta_local, ruta_remota)
            print(f"[+] Archivo subido: {ruta_local} → {ruta_remota}")
        elif direccion == "descargar":
            Path(ruta_local).parent.mkdir(parents=True, exist_ok=True)
            sftp.get(ruta_remota, ruta_local)
            print(f"[+] Archivo descargado: {ruta_remota} → {ruta_local}")

        sftp.close()
        return True

    except FileNotFoundError as e:
        print(f"[!] Archivo no encontrado: {e}")
        return False
    except Exception as e:
        print(f"[!] Error SFTP: {e}")
        return False


if __name__ == "__main__":
    HOST       = sys.argv[1] if len(sys.argv) > 1 else "192.168.56.102"
    USUARIO    = "msfadmin"
    CLAVE      = os.path.expanduser("~/.ssh/lab06_key")
    LOCAL_SRC  = "/root/lab06/resultados/enumeracion_ssh.txt"
    REMOTO_DST = "/tmp/enumeracion_desde_kali.txt"
    LOCAL_DST  = "/root/lab06/resultados/archivo_recuperado.txt"

    cliente = conectar_ssh_clave(HOST, 22, USUARIO, CLAVE)

    if cliente:
        print("\n[*] Probando transferencia SFTP...")

        # 1. Subir el archivo de enumeración a Metasploitable
        exito_subida = transferir_archivo_sftp(
            cliente, LOCAL_SRC, REMOTO_DST, "subir"
        )

        if exito_subida:
            # 2. Verificar que el archivo existe en el servidor remoto
            stdin, stdout, _ = cliente.exec_command(f"ls -lh {REMOTO_DST}")
            print(f"[*] Verificación remota: {stdout.read().decode().strip()}")

            # 3. Descargar el archivo de vuelta (simula recuperación de evidencia)
            transferir_archivo_sftp(
                cliente, LOCAL_DST, REMOTO_DST, "descargar"
            )

        cliente.close()
        print("[*] Sesión SSH cerrada.")
```

4. Ejecutar el script:

```bash
python3 parte_b/ssh_clave_sftp.py $METASPLOITABLE_IP
```

#### Salida esperada

```
[*] Clave privada cargada: /root/.ssh/lab06_key
[*] Conectando a 192.168.56.102:22 con clave pública...
[+] Autenticación por clave pública exitosa en 192.168.56.102

[*] Probando transferencia SFTP...
[+] Archivo subido: /root/lab06/resultados/enumeracion_ssh.txt → /tmp/enumeracion_desde_kali.txt
[*] Verificación remota: -rw-r--r-- 1 msfadmin msfadmin 2.1K ... /tmp/enumeracion_desde_kali.txt
[+] Archivo descargado: /tmp/enumeracion_desde_kali.txt → /root/lab06/resultados/archivo_recuperado.txt
[*] Sesión SSH cerrada.
```

#### Verificación

```bash
# Comparar el archivo original con el recuperado (deben ser idénticos)
diff ~/lab06/resultados/enumeracion_ssh.txt ~/lab06/resultados/archivo_recuperado.txt
echo "Diferencias: $? (0 = idénticos)"
```

---

### ─── INTEGRACIÓN: Scapy + Paramiko en Flujo Unificado ───

---

### Paso 6 — Script Integrador: Descubrimiento → Verificación SSH → Conexión Automática

**Objetivo:** Combinar el ARP scanner (Parte A) con la automatización SSH (Parte B) en un único script orquestador que descubra hosts, verifique si tienen SSH abierto usando `connect_ex()` (Lección 6.1), y se conecte automáticamente para ejecutar enumeración.

#### Instrucciones

1. Crear el archivo `~/lab06/integrado/toolkit_recon.py`:

```python
#!/usr/bin/env python3
"""
toolkit_recon.py — Toolkit Integrado: Scapy ARP Scan + Verificación SSH + Paramiko
Lab 06-00-01 | Integración Final
ADVERTENCIA: Ejecutar ÚNICAMENTE en la red interna de laboratorio con autorización.

Flujo de trabajo:
  1. ARP Scan (Scapy)          → Descubrir hosts activos en la red
  2. Verificación de puerto 22  → socket.connect_ex() por cada host (Lección 6.1)
  3. SYN Scan puertos clave     → Scapy (hosts con SSH abierto)
  4. Conexión SSH automática    → Paramiko con credenciales proporcionadas
  5. Enumeración y guardado     → Resultados en /resultados/
"""

import os
import sys
import socket
import json
import time
from pathlib import Path
from datetime import datetime

# Importaciones de Scapy
from scapy.all import ARP, Ether, IP, TCP, srp, sr1, send, conf as scapy_conf

# Importaciones de Paramiko
import paramiko

# Silenciar Scapy
scapy_conf.verb = 0

# ─── Configuración ────────────────────────────────────────────────────────────

CONFIG = {
    "red_objetivo":    os.environ.get("TARGET_NETWORK", "192.168.56.0/24"),
    "interfaz":        os.environ.get("IFACE", "eth1"),
    "puerto_ssh":      22,
    "usuario_ssh":     "msfadmin",
    "password_ssh":    "msfadmin",
    "clave_ssh":       os.path.expanduser("~/.ssh/lab06_key"),
    "timeout_arp":     2.0,
    "timeout_socket":  1.5,
    "timeout_ssh":     10.0,
    "directorio_out":  "/root/lab06/resultados",
    "puertos_scan":    [21, 22, 23, 25, 80, 139, 445, 3306, 5432, 8180],
}

Path(CONFIG["directorio_out"]).mkdir(parents=True, exist_ok=True)

# ─── Módulo 1: ARP Scan ───────────────────────────────────────────────────────

def arp_scan(red: str, iface: str, timeout: float) -> list[dict]:
    """Descubre hosts activos mediante ARP broadcast."""
    print(f"\n{'═'*60}")
    print(f"  [MÓDULO 1] ARP SCAN — {red}")
    print(f"{'═'*60}")

    paquete = Ether(dst="ff:ff:ff:ff:ff:ff") / ARP(pdst=red, op=1)
    respondidos, _ = srp(paquete, iface=iface, timeout=timeout, retry=1)

    hosts = [{"ip": r[ARP].psrc, "mac": r[Ether].src} for _, r in respondidos]

    for h in hosts:
        print(f"  [ARP] {h['ip']:<18} MAC: {h['mac']}")
    print(f"  → {len(hosts)} hosts descubiertos")
    return hosts

# ─── Módulo 2: Verificación de Puerto SSH ────────────────────────────────────

def verificar_ssh(ip: str, puerto: int, timeout: float) -> bool:
    """
    Verifica si el puerto SSH está abierto usando connect_ex() (Lección 6.1).
    Más liviano que un SYN scan completo para una verificación rápida.
    """
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.settimeout(timeout)
        try:
            resultado = s.connect_ex((ip, puerto))
            return resultado == 0
        except (socket.gaierror, OSError):
            return False

# ─── Módulo 3: SYN Scan de Puertos Clave ─────────────────────────────────────

def syn_scan_host(host: str, puertos: list[int], timeout: float = 1.0) -> list[int]:
    """Retorna lista de puertos abiertos en el host objetivo."""
    import random
    abiertos = []
    for puerto in puertos:
        po = random.randint(1024, 65535)
        pkt = IP(dst=host) / TCP(sport=po, dport=puerto, flags="S",
                                  seq=random.randint(0, 2**32 - 1))
        resp = sr1(pkt, timeout=timeout, verbose=0)
        if resp and resp.haslayer(TCP) and resp[TCP].flags == "SA":
            abiertos.append(puerto)
            rst = IP(dst=host) / TCP(sport=po, dport=puerto, flags="R",
                                      seq=resp[TCP].ack)
            send(rst, verbose=0)
    return abiertos

# ─── Módulo 4: Enumeración SSH ────────────────────────────────────────────────

def enumerar_host_ssh(
    ip: str, usuario: str, password: str,
    clave_path: str, timeout: float
) -> dict | None:
    """
    Intenta conectar por SSH (primero clave, luego contraseña) y enumera el host.
    """
    cliente = paramiko.SSHClient()
    cliente.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    metodo_auth = None

    # Intentar primero con clave pública
    if Path(clave_path).exists():
        try:
            clave = paramiko.RSAKey.from_private_key_file(clave_path)
            cliente.connect(ip, port=22, username=usuario, pkey=clave,
                            timeout=timeout, look_for_keys=False, allow_agent=False)
            metodo_auth = "clave_publica"
        except Exception:
            pass

    # Si falla la clave, intentar con contraseña
    if metodo_auth is None:
        try:
            cliente.connect(ip, port=22, username=usuario, password=password,
                            timeout=timeout, look_for_keys=False, allow_agent=False)
            metodo_auth = "password"
        except Exception as e:
            print(f"  [SSH] No se pudo conectar a {ip}: {e}")
            return None

    print(f"  [SSH] Conectado a {ip} (auth: {metodo_auth})")

    # Ejecutar comandos de enumeración
    comandos = {
        "os":      "uname -a",
        "usuario": "id",
        "red":     "ifconfig 2>/dev/null || ip addr show | head -20",
        "uptime":  "uptime",
    }

    datos = {"ip": ip, "auth": metodo_auth, "timestamp": datetime.now().isoformat()}
    for clave_cmd, cmd in comandos.items():
        stdin, stdout, stderr = cliente.exec_command(cmd)
        datos[clave_cmd] = stdout.read().decode("utf-8", errors="replace").strip()

    cliente.close()
    return datos

# ─── Orquestador Principal ────────────────────────────────────────────────────

def main():
    inicio = time.time()
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    reporte_final = []

    print(f"\n{'█'*60}")
    print(f"  TOOLKIT DE RECONOCIMIENTO — Lab 06-00-01")
    print(f"  Inicio: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print(f"{'█'*60}")

    # ── FASE 1: Descubrimiento ARP ──────────────────────────────────────────
    hosts = arp_scan(CONFIG["red_objetivo"], CONFIG["interfaz"], CONFIG["timeout_arp"])

    if not hosts:
        print("[!] No se descubrieron hosts. Verificar configuración de red.")
        sys.exit(1)

    # ── FASE 2: Filtrar hosts con SSH abierto ───────────────────────────────
    print(f"\n{'═'*60}")
    print(f"  [MÓDULO 2] VERIFICACIÓN SSH (puerto {CONFIG['puerto_ssh']})")
    print(f"{'═'*60}")

    hosts_con_ssh = []
    for host in hosts:
        tiene_ssh = verificar_ssh(
            host["ip"], CONFIG["puerto_ssh"], CONFIG["timeout_socket"]
        )
        estado = "SSH ABIERTO ✓" if tiene_ssh else "SSH CERRADO ✗"
        print(f"  {host['ip']:<18} {estado}")
        if tiene_ssh:
            hosts_con_ssh.append(host)

    print(f"  → {len(hosts_con_ssh)} hosts con SSH disponible")

    # ── FASE 3: SYN Scan sobre hosts con SSH ───────────────────────────────
    print(f"\n{'═'*60}")
    print(f"  [MÓDULO 3] SYN SCAN — Puertos clave")
    print(f"{'═'*60}")

    for host in hosts_con_ssh:
        print(f"\n  Escaneando {host['ip']}...")
        puertos_abiertos = syn_scan_host(
            host["ip"], CONFIG["puertos_scan"]
        )
        host["puertos_abiertos"] = puertos_abiertos
        print(f"  Puertos abiertos: {puertos_abiertos if puertos_abiertos else 'ninguno'}")

    # ── FASE 4: Enumeración SSH ─────────────────────────────────────────────
    print(f"\n{'═'*60}")
    print(f"  [MÓDULO 4] ENUMERACIÓN SSH AUTOMÁTICA")
    print(f"{'═'*60}")

    for host in hosts_con_ssh:
        print(f"\n  → Enumerando {host['ip']}...")
        datos = enumerar_host_ssh(
            host["ip"],
            CONFIG["usuario_ssh"],
            CONFIG["password_ssh"],
            CONFIG["clave_ssh"],
            CONFIG["timeout_ssh"]
        )
        if datos:
            datos["puertos_abiertos"] = host.get("puertos_abiertos", [])
            reporte_final.append(datos)
            print(f"  OS: {datos.get('os', 'N/A')[:80]}")
            print(f"  Usuario: {datos.get('usuario', 'N/A')}")

    # ── GUARDAR REPORTE ─────────────────────────────────────────────────────
    archivo_reporte = f"{CONFIG['directorio_out']}/reporte_integrado_{timestamp}.json"
    with open(archivo_reporte, "w") as f:
        json.dump(reporte_final, f, indent=2, ensure_ascii=False)

    duracion = time.time() - inicio
    print(f"\n{'█'*60}")
    print(f"  TOOLKIT COMPLETADO en {duracion:.1f} segundos")
    print(f"  Hosts enumerados: {len(reporte_final)}")
    print(f"  Reporte guardado: {archivo_reporte}")
    print(f"{'█'*60}\n")


if __name__ == "__main__":
    main()
```

2. Ejecutar el toolkit integrador (requiere `sudo` por Scapy):

```bash
cd ~/lab06
source venv/bin/activate
sudo -E python3 integrado/toolkit_recon.py
```

> **Nota:** El flag `-E` de `sudo` preserva las variables de entorno (`TARGET_NETWORK`, `IFACE`, `METASPLOITABLE_IP`) definidas en la Sección 5.4.

3. Revisar el reporte JSON generado:

```bash
# Ver el reporte con formato legible
python3 -m json.tool ~/lab06/resultados/reporte_integrado_*.json | head -60
```

#### Salida esperada

```
████████████████████████████████████████████████████████████
  TOOLKIT DE RECONOCIMIENTO — Lab 06-00-01
  Inicio: 2024-11-15 14:35:00
████████████████████████████████████████████████████████████

══════════════════════════════════════════════════════════════
  [MÓDULO 1] ARP SCAN — 192.168.56.0/24
══════════════════════════════════════════════════════════════
  [ARP] 192.168.56.1        MAC: 0a:00:27:00:00:00
  [ARP] 192.168.56.101      MAC: 08:00:27:ab:cd:ef
  [ARP] 192.168.56.102      MAC: 08:00:27:12:34:56
  → 3 hosts descubiertos

══════════════════════════════════════════════════════════════
  [MÓDULO 2] VERIFICACIÓN SSH (puerto 22)
══════════════════════════════════════════════════════════════
  192.168.56.1       SSH CERRADO ✗
  192.168.56.101     SSH CERRADO ✗
  192.168.56.102     SSH ABIERTO ✓
  → 1 hosts con SSH disponible

  Escaneando 192.168.56.102...
  Puertos abiertos: [21, 22, 23, 25, 80, 139, 445, 3306, 5432, 8180]

  → Enumerando 192.168.56.102...
  [SSH] Conectado a 192.168.56.102 (auth: clave_publica)
  OS: Linux metasploitable 2.6.24-16-server #1 SMP...
  Usuario: uid=1000(msfadmin) gid=1000(msfadmin)...

████████████████████████████████████████████████████████████
  TOOLKIT COMPLETADO en 18.4 segundos
  Hosts enumerados: 1
  Reporte guardado: /root/lab06/resultados/reporte_integrado_20241115_143518.json
████████████████████████████████████████████████████████████
```

#### Verificación

```bash
# Verificar que el reporte JSON es válido y contiene datos de enumeración
python3 -c "
import json, glob
archivos = glob.glob('/root/lab06/resultados/reporte_integrado_*.json')
with open(sorted(archivos)[-1]) as f:
    datos = json.load(f)
print(f'Hosts en reporte: {len(datos)}')
for h in datos:
    print(f'  IP: {h[\"ip\"]} | OS: {h.get(\"os\",\"N/A\")[:50]}')
    print(f'  Puertos: {h.get(\"puertos_abiertos\",[])}')
"
```

---

## 7. Validación y Pruebas

Ejecutar la siguiente suite de verificación al finalizar todos los pasos:

```bash
#!/usr/bin/env bash
# validate_lab06.sh — Suite de validación del Lab 06-00-01

echo "══════════════════════════════════════════════════"
echo "  VALIDACIÓN LAB 06-00-01"
echo "══════════════════════════════════════════════════"

PASS=0
FAIL=0

check() {
    local desc="$1"
    local cmd="$2"
    if eval "$cmd" &>/dev/null; then
        echo "  ✓ $desc"
        ((PASS++))
    else
        echo "  ✗ FALLO: $desc"
        ((FAIL++))
    fi
}

# ── Parte A: Scapy ────────────────────────────────────────────
echo ""
echo "  [Parte A] Scripts Scapy"
check "arp_scanner.py existe"      "test -f ~/lab06/parte_a/arp_scanner.py"
check "syn_scanner.py existe"      "test -f ~/lab06/parte_a/syn_scanner.py"
check "sniffer_pasivo.py existe"   "test -f ~/lab06/parte_a/sniffer_pasivo.py"
check "hosts_arp.csv generado"     "test -s ~/lab06/resultados/hosts_arp.csv"
check "CSV contiene cabecera ip,mac" "head -1 ~/lab06/resultados/hosts_arp.csv | grep -q 'ip,mac'"
check "captura_pasiva.csv generado" "test -s ~/lab06/resultados/captura_pasiva.csv"

# ── Parte B: Paramiko ─────────────────────────────────────────
echo ""
echo "  [Parte B] Scripts Paramiko"
check "ssh_password.py existe"     "test -f ~/lab06/parte_b/ssh_password.py"
check "ssh_clave_sftp.py existe"   "test -f ~/lab06/parte_b/ssh_clave_sftp.py"
check "enumeracion_ssh.txt generado" "test -s ~/lab06/resultados/enumeracion_ssh.txt"
check "archivo_recuperado.txt existe" "test -s ~/lab06/resultados/archivo_recuperado.txt"
check "Clave SSH generada"         "test -f ~/.ssh/lab06_key"
check "Enumeración contiene 'Linux'" "grep -qi 'linux' ~/lab06/resultados/enumeracion_ssh.txt"

# ── Integración ───────────────────────────────────────────────
echo ""
echo "  [Integración] Toolkit unificado"
check "toolkit_recon.py existe"    "test -f ~/lab06/integrado/toolkit_recon.py"
check "Reporte JSON generado"      "ls ~/lab06/resultados/reporte_integrado_*.json 2>/dev/null | grep -q json"
check "Reporte JSON válido"        "python3 -m json.tool ~/lab06/resultados/reporte_integrado_*.json > /dev/null 2>&1"

# ── Resumen ───────────────────────────────────────────────────
echo ""
echo "══════════════════════════════════════════════════"
echo "  Resultados: $PASS pasaron, $FAIL fallaron"
echo "══════════════════════════════════════════════════"
[ $FAIL -eq 0 ] && echo "  ✓ Laboratorio completado exitosamente" || echo "  ✗ Revisar los ítems fallidos"
```

```bash
# Ejecutar la suite de validación
chmod +x validate_lab06.sh
bash validate_lab06.sh
```

---

## 8. Resolución de Problemas

### Problema 1: Scapy no envía/recibe paquetes — Error de permisos o sin respuestas

**Síntomas:**
```
WARNING: No route found for IPv6 destination :: (no default route?)
PermissionError: [Errno 1] Operation not permitted
# O bien: el ARP scan retorna 0 hosts aunque Metasploitable 2 esté activa
```

**Causa:**
Scapy requiere acceso a raw sockets para construir paquetes a nivel de capa 2/3. Si el script se ejecuta sin `sudo`, el kernel deniega la operación. Adicionalmente, si la interfaz especificada no es la correcta (por ejemplo, se usa `eth0` cuando la red interna está en `eth1`), los paquetes ARP se envían por la interfaz equivocada y no reciben respuesta.

**Solución:**
```bash
# 1. Verificar siempre la interfaz correcta antes de ejecutar
ip addr show
ip route show
# Identificar la interfaz cuya IP pertenece al rango de la red interna

# 2. Ejecutar con sudo y pasar la interfaz explícitamente
sudo python3 parte_a/arp_scanner.py 192.168.56.0/24 eth1
#                                                        ^^^^ ajustar

# 3. Si el error persiste, verificar que la tarjeta de red tiene modo promiscuo
sudo ip link set eth1 promisc on
ip link show eth1 | grep PROMISC

# 4. Verificar que Scapy puede listar interfaces disponibles
sudo python3 -c "from scapy.all import get_if_list; print(get_if_list())"

# 5. Probar con un ping manual para confirmar conectividad básica
ping -c 2 $METASPLOITABLE_IP
```

---

### Problema 2: Paramiko falla con `AuthenticationException` al usar clave pública

**Síntomas:**
```
paramiko.ssh_exception.AuthenticationException: Authentication failed.
# O bien:
[!] Autenticación fallida: clave pública no autorizada en el servidor
```

**Causa:**
La clave pública no fue copiada correctamente al archivo `~/.ssh/authorized_keys` de Metasploitable 2, o los permisos del directorio `~/.ssh` o del archivo `authorized_keys` en el servidor remoto son incorrectos (SSH rechaza claves si los permisos son demasiado permisivos). También puede ocurrir si la clave privada local no corresponde a la pública instalada en el servidor.

**Solución:**
```bash
# 1. Verificar que la clave pública está en el servidor remoto
ssh -i ~/.ssh/lab06_key msfadmin@$METASPLOITABLE_IP \
  "cat ~/.ssh/authorized_keys"
# La clave pública de lab06_key.pub debe aparecer en la lista

# 2. Corregir permisos en el servidor remoto (conectar primero con contraseña)
ssh msfadmin@$METASPLOITABLE_IP << 'EOF'
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
ls -la ~/.ssh/
EOF

# 3. Verificar que la clave privada local tiene permisos correctos
chmod 600 ~/.ssh/lab06_key
ls -la ~/.ssh/lab06_key

# 4. Probar la clave manualmente con el cliente SSH nativo
ssh -i ~/.ssh/lab06_key -v msfadmin@$METASPLOITABLE_IP "id"
# El flag -v muestra el proceso de autenticación en detalle

# 5. Si la clave fue regenerada, reinstalar la clave pública
ssh-copy-id -i ~/.ssh/lab06_key.pub msfadmin@$METASPLOITABLE_IP
```

---

## 9. Limpieza del Entorno

Ejecutar los siguientes pasos al finalizar el laboratorio para restaurar el entorno a un estado limpio:

```bash
# ── 1. Deshabilitar modo promiscuo en la interfaz ──────────────────────────
sudo ip link set $IFACE promisc off
ip link show $IFACE | grep -v PROMISC && echo "✓ Modo promiscuo deshabilitado"

# ── 2. Eliminar la clave pública del servidor Metasploitable 2 ─────────────
# Esto es importante para no dejar acceso persistente en la VM
ssh msfadmin@$METASPLOITABLE_IP << 'EOF'
# Eliminar la clave del laboratorio de authorized_keys
grep -v "lab06-kali" ~/.ssh/authorized_keys > /tmp/auth_tmp && \
  mv /tmp/auth_tmp ~/.ssh/authorized_keys
echo "Claves restantes en authorized_keys:"
cat ~/.ssh/authorized_keys | wc -l
EOF

# ── 3. Eliminar archivos temporales del servidor remoto ────────────────────
ssh msfadmin@$METASPLOITABLE_IP "rm -f /tmp/enumeracion_desde_kali.txt"

# ── 4. Archivar resultados del laboratorio ─────────────────────────────────
FECHA=$(date +%Y%m%d_%H%M%S)
tar -czf ~/lab06_resultados_$FECHA.tar.gz ~/lab06/resultados/
echo "✓ Resultados archivados en: ~/lab06_resultados_$FECHA.tar.gz"

# ── 5. Desactivar el entorno virtual ──────────────────────────────────────
deactivate
echo "✓ Entorno virtual desactivado"

# ── 6. Restaurar snapshot de Metasploitable 2 (RECOMENDADO) ───────────────
# Apagar Metasploitable 2 primero, luego:
VBoxManage snapshot "Metasploitable2" restore "Lab06_inicio_limpio"
echo "✓ Snapshot restaurado: Lab06_inicio_limpio"

# ── 7. Verificar que no quedan procesos de Scapy activos ──────────────────
pgrep -a python3 | grep -i "scapy\|sniffer\|scanner" && \
  echo "⚠ Procesos activos detectados — terminar manualmente" || \
  echo "✓ Sin procesos activos de Scapy"
```

---

## 10. Resumen

### Conceptos Practicados

En este laboratorio construiste un toolkit de reconocimiento modular en Python que integra dos bibliotecas especializadas de seguridad:

| Módulo | Herramienta | Técnica aplicada | Resultado |
|--------|-------------|------------------|-----------|
| ARP Scanner | Scapy (`ARP`, `Ether`, `srp`) | Broadcast ARP en capa 2 | Descubrimiento de hosts activos |
| SYN Scanner | Scapy (`IP`, `TCP`, `sr1`) | Half-open scan (SYN sin ACK) | Puertos abiertos sin handshake completo |
| Sniffer pasivo | Scapy (`sniff`, callbacks) | Captura promiscua clasificada | Análisis de tráfico en tiempo real |
| SSH por contraseña | Paramiko (`SSHClient`, `exec_command`) | Autenticación password | Enumeración remota automatizada |
| SSH por clave + SFTP | Paramiko (`RSAKey`, `open_sftp`) | PKI + transferencia segura | Recuperación de evidencia |
| Toolkit integrado | Scapy + `socket` + Paramiko | Flujo ARP→SSH→enum | Reporte JSON unificado |

### Conexión con la Lección 6.1

El uso de `socket.connect_ex()` en el módulo de verificación SSH del toolkit integrador aplica directamente el patrón de la Lección 6.1: crear un socket TCP, establecer timeout con `settimeout()`, y usar el código de retorno de `connect_ex()` para determinar si el puerto está abierto sin lanzar excepciones. Esta técnica liviana complementa el SYN scan de Scapy para una verificación rápida previa.

### Principios Éticos Aplicados

- Todos los scripts incluyen advertencias de uso responsable y están limitados a la red interna.
- El SYN scanner envía RST inmediatamente tras recibir SYN-ACK para no completar conexiones innecesarias.
- Las credenciales SSH no están hardcodeadas en variables de entorno exportables a producción.
- La clave pública instalada en Metasploitable 2 fue eliminada en la fase de limpieza.

### Recursos Adicionales

- [Documentación oficial de Scapy](https://scapy.readthedocs.io/en/latest/)
- [Documentación oficial de Paramiko](https://www.paramiko.org/api/index.html)
- [Scapy — Sending and Receiving Packets](https://scapy.readthedocs.io/en/latest/usage.html#send-and-receive-packets-sr)
- [RFC 793: TCP — Especificación del handshake de tres vías](https://www.rfc-editor.org/rfc/rfc793)
- [Python socket.connect_ex() — Documentación oficial](https://docs.python.org/3/library/socket.html#socket.socket.connect_ex)
- [Paramiko — SSHClient.exec_command()](https://docs.paramiko.org/en/stable/api/client.html#paramiko.client.SSHClient.exec_command)

---
*Lab 06-00-01 — Práctica 6: Scripts de Scapy y Rutinas SSH
