# Práctica 7 — Integración Práctica con Metasploit y TOR

## 1. Metadatos

| Campo            | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 46 minutos                                   |
| **Complejidad**  | Difícil                                      |
| **Nivel Bloom**  | Crear                                        |
| **Laboratorio**  | 07-00-01                                     |
| **Módulo**       | Práctica 7: Integración Metasploit + TOR     |

---

## 2. Descripción General

En esta práctica integrarás dos tecnologías avanzadas dentro de tu toolkit de hacking ético: el control programático de Metasploit Framework a través de su interfaz RPC usando `pymetasploit3`, y el enrutamiento anónimo de tráfico HTTP a través de la red TOR usando `stem` y `PySocks`. Automatizarás módulos auxiliares no destructivos de Metasploit contra Metasploitable 2, verificarás la conectividad TOR desde Python, implementarás rotación de circuito, y finalmente refactorizarás ambas funcionalidades como módulos importables y reutilizables. Todo el trabajo se ejecuta en red interna aislada con autorización documentada.

---

## 3. Objetivos de Aprendizaje

- [ ] Conectar y controlar Metasploit Framework desde Python usando `pymetasploit3` vía MSFRPC, iniciando `msfrpcd` y ejecutando llamadas RPC estructuradas.
- [ ] Automatizar la ejecución de módulos auxiliares no destructivos (`auxiliary/scanner/portscan/tcp` y `auxiliary/scanner/http/http_version`) contra Metasploitable 2, recuperando y mostrando sus resultados.
- [ ] Configurar el daemon TOR, verificar la conexión con `stem`, enrutar peticiones HTTP a través de SOCKS5 e implementar rotación de circuito desde Python.
- [ ] Diseñar y entregar dos módulos Python importables (`msf_module.py` y `tor_module.py`) con interfaces limpias y controles de error apropiados para el toolkit final.

---

## 4. Prerrequisitos

### Conocimiento Previo
- Haber completado **Lab 04-00-01** (reconocimiento con Scapy) y **Lab 05-00-01** (automatización con Requests/Paramiko).
- Comprender el modelo cliente-servidor y el concepto de llamadas RPC.
- Familiaridad básica con Metasploit Framework (`msfconsole`, módulos auxiliares).
- Conocimiento de proxies SOCKS5 y el modelo de circuitos de TOR.

### Acceso y Autorización
- **Formulario de autorización firmado** (proporcionado por el instructor) para Labs 4–8. Sin este documento no se puede continuar.
- Kali Linux 2024.1+ como máquina atacante con Metasploit Framework 6.3+ instalado.
- Metasploitable 2 activa y accesible en red NAT interna aislada de Internet.
- Snapshot reciente de Metasploitable 2 creado **antes** de iniciar este laboratorio.

---

## 5. Entorno de Laboratorio

### Hardware Requerido

| Componente        | Mínimo                                      |
|-------------------|---------------------------------------------|
| RAM               | 8 GB (16 GB recomendado)                    |
| CPU               | 64-bit, 4 núcleos, virtualización habilitada|
| Disco libre       | 60 GB                                       |
| Red               | Tarjeta compatible con modo promiscuo       |
| Pantalla          | 1280×768 mínimo                             |

### Software Requerido

| Software                | Versión mínima | Máquina         |
|-------------------------|----------------|-----------------|
| Kali Linux              | 2024.1+        | VM Atacante     |
| Metasploit Framework    | 6.3+           | Kali Linux      |
| Python                  | 3.10+          | Kali Linux      |
| pymetasploit3           | última estable | Kali Linux      |
| stem                    | 1.8+           | Kali Linux      |
| PySocks                 | 1.7+           | Kali Linux      |
| requests                | 2.31+          | Kali Linux      |
| tor (daemon)            | última estable | Kali Linux      |
| Metasploitable 2        | —              | VM Objetivo     |

### Topología de Red

```
┌─────────────────────┐         Red NAT Interna (AISLADA)         ┌──────────────────────┐
│   Kali Linux        │◄──────────────────────────────────────────►│  Metasploitable 2    │
│   192.168.56.10     │         (sin acceso a Internet)            │  192.168.56.20       │
│   (VM Atacante)     │                                            │  (VM Objetivo)       │
└─────────────────────┘                                            └──────────────────────┘
```

> **⚠️ AISLAMIENTO OBLIGATORIO:** Verificar que la red NAT interna **no tenga salida a Internet** antes de iniciar cualquier escaneo activo. Metasploitable 2 nunca debe tener acceso a redes externas.

### Comandos de Preparación del Entorno

**Paso 0 — Antes de comenzar: crear snapshot de Metasploitable 2**

En VirtualBox, antes de iniciar la VM objetivo:
```bash
# Desde la terminal del host (no dentro de la VM)
VBoxManage snapshot "Metasploitable2" take "Pre-Lab07" --description "Estado limpio antes de Lab 07-00-01"
```

**Paso 0b — Verificar conectividad de red (desde Kali)**
```bash
# Confirmar IP de Kali en la red interna
ip addr show eth1

# Verificar que Metasploitable 2 responde
ping -c 3 192.168.56.20

# Confirmar que NO hay salida a Internet desde la red interna
ping -c 2 8.8.8.8  # Debe fallar o no obtener respuesta
```

**Paso 0c — Instalar dependencias Python**
```bash
# Activar entorno virtual del toolkit (creado en labs anteriores)
cd ~/eth-toolkit
source venv/bin/activate

# Instalar bibliotecas necesarias para este laboratorio
pip install pymetasploit3 PySocks stem requests

# Verificar instalaciones
python -c "from pymetasploit3.msfrpc import MsfRpcClient; print('[OK] pymetasploit3')"
python -c "import socks; print('[OK] PySocks')"
python -c "import stem; print('[OK] stem')"
```

**Paso 0d — Instalar y habilitar TOR**
```bash
# Instalar TOR si no está presente
sudo apt update && sudo apt install -y tor

# Verificar que TOR está instalado
tor --version

# Iniciar el daemon TOR
sudo systemctl start tor
sudo systemctl status tor
```

---

## 6. Procedimiento Paso a Paso

---

### PARTE A — Control de Metasploit desde Python

---

### Paso A1: Iniciar el Servidor MSFRPC

**Objetivo:** Levantar el daemon RPC de Metasploit para que Python pueda conectarse a él mediante llamadas HTTP.

**Instrucciones:**

1. Abre una terminal dedicada en Kali Linux. Esta terminal quedará ocupada por `msfrpcd`.

2. Inicia el servidor RPC con credenciales definidas:
```bash
# -P: contraseña  -U: usuario  -p: puerto  -S: sin SSL (solo laboratorio local aislado)
msfrpcd -P Lab07Pass! -U msf -p 55553 -S
```

3. Espera aproximadamente 10–15 segundos a que el servidor inicialice. Verás un mensaje similar a:
```
[*] MSGRPC starting on 0.0.0.0:55553 (SSL:false)...
[*] MSGRPC ready at 2024-xx-xx xx:xx:xx +0000.
```

4. **No cierres esta terminal.** Abre una segunda terminal para el trabajo con Python.

5. Verifica que el puerto está escuchando:
```bash
ss -tlnp | grep 55553
```

**Salida esperada:**
```
LISTEN  0  128  0.0.0.0:55553  0.0.0.0:*  users:(("ruby",pid=XXXX,fd=XX))
```

**Verificación:**
```bash
# Desde la segunda terminal, confirmar conectividad al puerto RPC
nc -zv 127.0.0.1 55553
# Debe mostrar: Connection to 127.0.0.1 55553 port [tcp/*] succeeded!
```

> **⚠️ Seguridad:** La opción `-S` desactiva SSL y **solo** es aceptable en este laboratorio local completamente aislado. En entornos reales, omitir `-S` para forzar TLS.

---

### Paso A2: Conectar Python a MSFRPC y Listar Módulos

**Objetivo:** Establecer la conexión desde Python, verificar que funciona, y construir una función reutilizable para listar módulos auxiliares por término de búsqueda.

**Instrucciones:**

1. En el directorio del toolkit, crea el archivo de trabajo inicial:
```bash
cd ~/eth-toolkit
mkdir -p lab07
touch lab07/msf_test.py
```

2. Escribe el siguiente script en `lab07/msf_test.py`:

```python
#!/usr/bin/env python3
"""
Lab 07-00-01 — Parte A: Conexión y exploración de módulos en Metasploit RPC
ADVERTENCIA: Ejecutar ÚNICAMENTE en entornos de laboratorio autorizados.
"""

from pymetasploit3.msfrpc import MsfRpcClient
import time
import sys

# ─────────────────────────────────────────────────────────────
# Configuración de conexión RPC
# ─────────────────────────────────────────────────────────────
MSF_HOST     = '127.0.0.1'
MSF_PORT     = 55553
MSF_USER     = 'msf'
MSF_PASSWORD = 'Lab07Pass!'
MSF_SSL      = False   # Solo laboratorio local aislado


def conectar_msf() -> MsfRpcClient:
    """
    Establece y devuelve una conexión al servidor MSFRPC.
    Lanza SystemExit si la conexión falla.
    """
    try:
        client = MsfRpcClient(
            password=MSF_PASSWORD,
            username=MSF_USER,
            port=MSF_PORT,
            ssl=MSF_SSL,
            server=MSF_HOST
        )
        print("[+] Conexión establecida con Metasploit RPC")
        print(f"    Host : {MSF_HOST}:{MSF_PORT}")
        print(f"    SSL  : {MSF_SSL}")
        return client
    except Exception as e:
        print(f"[-] Error al conectar con MSFRPC: {e}", file=sys.stderr)
        print("    ¿Está msfrpcd ejecutándose? Verifica con: ss -tlnp | grep 55553")
        sys.exit(1)


def listar_modulos_auxiliares(client: MsfRpcClient, termino: str) -> list[dict]:
    """
    Busca módulos auxiliares que coincidan con 'termino'.
    Devuelve lista de diccionarios con 'fullname', 'type', 'name'.
    """
    print(f"\n[*] Buscando módulos auxiliares con término: '{termino}'")
    resultados = client.modules.search(termino)
    auxiliares = [m for m in resultados if m.get('type') == 'auxiliary']
    print(f"[*] Módulos auxiliares encontrados: {len(auxiliares)}")
    for mod in auxiliares[:10]:
        print(f"    - {mod.get('fullname', 'N/A')}")
    return auxiliares


# ─────────────────────────────────────────────────────────────
# Ejecución principal
# ─────────────────────────────────────────────────────────────
if __name__ == '__main__':
    client = conectar_msf()

    # Listar módulos de escaneo de puertos
    listar_modulos_auxiliares(client, 'portscan')

    # Listar módulos de versión HTTP
    listar_modulos_auxiliares(client, 'http_version')
```

3. Ejecuta el script:
```bash
python lab07/msf_test.py
```

**Salida esperada:**
```
[+] Conexión establecida con Metasploit RPC
    Host : 127.0.0.1:55553
    SSL  : False

[*] Buscando módulos auxiliares con término: 'portscan'
[*] Módulos auxiliares encontrados: 5
    - auxiliary/scanner/portscan/tcp
    - auxiliary/scanner/portscan/syn
    - auxiliary/scanner/portscan/ack
    - auxiliary/scanner/portscan/ftpbounce
    - auxiliary/scanner/portscan/xmas

[*] Buscando módulos auxiliares con término: 'http_version'
[*] Módulos auxiliares encontrados: 1
    - auxiliary/scanner/http/http_version
```

**Verificación:**
- La salida muestra al menos `auxiliary/scanner/portscan/tcp` y `auxiliary/scanner/http/http_version`.
- No aparece ningún error de conexión rechazada.

---

### Paso A3: Automatizar el Módulo `auxiliary/scanner/portscan/tcp`

**Objetivo:** Configurar y ejecutar el escáner de puertos TCP de Metasploit contra Metasploitable 2 de forma completamente automatizada desde Python, recuperando los resultados del trabajo.

**Instrucciones:**

1. Crea el archivo `lab07/msf_scanner.py`:

```python
#!/usr/bin/env python3
"""
Lab 07-00-01 — Parte A: Automatización de módulos auxiliares de Metasploit
Módulos usados: auxiliary/scanner/portscan/tcp
                auxiliary/scanner/http/http_version
OBJETIVO AUTORIZADO: Metasploitable 2 en red interna de laboratorio.
"""

from pymetasploit3.msfrpc import MsfRpcClient
import time
import sys

# ─────────────────────────────────────────────────────────────
# Configuración
# ─────────────────────────────────────────────────────────────
MSF_HOST      = '127.0.0.1'
MSF_PORT      = 55553
MSF_USER      = 'msf'
MSF_PASSWORD  = 'Lab07Pass!'
MSF_SSL       = False

TARGET_IP     = '192.168.56.20'   # Metasploitable 2 — SOLO objetivo autorizado
THREADS       = 4
TCP_PORTS     = '22,80,139,445,3306,8180'   # Puertos conocidos de Metasploitable 2
JOB_WAIT_SEC  = 20                           # Segundos máximos de espera por trabajo


def conectar_msf() -> MsfRpcClient:
    """Establece conexión con MSFRPC. Lanza SystemExit si falla."""
    try:
        client = MsfRpcClient(
            password=MSF_PASSWORD,
            username=MSF_USER,
            port=MSF_PORT,
            ssl=MSF_SSL,
            server=MSF_HOST
        )
        print("[+] Conexión MSFRPC establecida")
        return client
    except Exception as e:
        print(f"[-] Error de conexión MSFRPC: {e}", file=sys.stderr)
        sys.exit(1)


def esperar_trabajo(client: MsfRpcClient, job_id, timeout: int = JOB_WAIT_SEC) -> bool:
    """
    Espera a que un trabajo de Metasploit finalice.
    Devuelve True si finalizó, False si superó el timeout.
    """
    print(f"[*] Esperando finalización del Job ID: {job_id} (timeout: {timeout}s)")
    inicio = time.time()
    while time.time() - inicio < timeout:
        jobs_activos = client.jobs.list
        if str(job_id) not in jobs_activos:
            print(f"[+] Job {job_id} finalizado correctamente.")
            return True
        time.sleep(2)
        print(f"    ... job {job_id} aún en ejecución ({int(time.time()-inicio)}s)")
    print(f"[!] Timeout alcanzado para Job {job_id}. Puede seguir en background.")
    return False


def ejecutar_portscan_tcp(client: MsfRpcClient, rhosts: str, ports: str, threads: int) -> dict:
    """
    Ejecuta auxiliary/scanner/portscan/tcp contra rhosts.
    Devuelve el resultado del método execute().
    """
    print(f"\n{'='*60}")
    print(f"[*] MÓDULO: auxiliary/scanner/portscan/tcp")
    print(f"[*] TARGET: {rhosts}  PORTS: {ports}  THREADS: {threads}")
    print(f"{'='*60}")

    # Cargar el módulo
    modulo = client.modules.use('auxiliary', 'scanner/portscan/tcp')

    # Mostrar opciones requeridas
    print("[*] Opciones requeridas del módulo:")
    for nombre, detalles in modulo.required.items():
        print(f"    {nombre}: {detalles}")

    # Configurar opciones
    modulo['RHOSTS']  = rhosts
    modulo['PORTS']   = ports
    modulo['THREADS'] = threads
    print(f"\n[+] Opciones configuradas: RHOSTS={rhosts}, PORTS={ports}, THREADS={threads}")

    # Ejecutar módulo
    resultado = modulo.execute(payload='')
    job_id = resultado.get('job_id')
    print(f"[+] Módulo ejecutado. Job ID: {job_id}")

    # Esperar finalización
    esperar_trabajo(client, job_id, timeout=JOB_WAIT_SEC)

    return resultado


def ejecutar_http_version(client: MsfRpcClient, rhosts: str, threads: int) -> dict:
    """
    Ejecuta auxiliary/scanner/http/http_version contra rhosts.
    Devuelve el resultado del método execute().
    """
    print(f"\n{'='*60}")
    print(f"[*] MÓDULO: auxiliary/scanner/http/http_version")
    print(f"[*] TARGET: {rhosts}  THREADS: {threads}")
    print(f"{'='*60}")

    # Cargar el módulo
    modulo = client.modules.use('auxiliary', 'scanner/http/http_version')

    # Mostrar opciones requeridas
    print("[*] Opciones requeridas del módulo:")
    for nombre, detalles in modulo.required.items():
        print(f"    {nombre}: {detalles}")

    # Configurar opciones
    modulo['RHOSTS']  = rhosts
    modulo['THREADS'] = threads
    print(f"\n[+] Opciones configuradas: RHOSTS={rhosts}, THREADS={threads}")

    # Ejecutar módulo
    resultado = modulo.execute(payload='')
    job_id = resultado.get('job_id')
    print(f"[+] Módulo ejecutado. Job ID: {job_id}")

    # Esperar finalización
    esperar_trabajo(client, job_id, timeout=JOB_WAIT_SEC)

    return resultado


def mostrar_jobs_activos(client: MsfRpcClient) -> None:
    """Muestra los trabajos activos en Metasploit."""
    jobs = client.jobs.list
    if jobs:
        print(f"\n[*] Jobs activos en Metasploit: {len(jobs)}")
        for jid, jinfo in jobs.items():
            print(f"    Job {jid}: {jinfo.get('name', 'N/A')}")
    else:
        print("\n[*] No hay jobs activos en Metasploit.")


# ─────────────────────────────────────────────────────────────
# Ejecución principal
# ─────────────────────────────────────────────────────────────
if __name__ == '__main__':
    client = conectar_msf()

    # Verificar estado inicial de jobs
    mostrar_jobs_activos(client)

    # Ejecutar escáner TCP
    res_tcp = ejecutar_portscan_tcp(
        client  = client,
        rhosts  = TARGET_IP,
        ports   = TCP_PORTS,
        threads = THREADS
    )

    # Pequeña pausa entre módulos
    time.sleep(3)

    # Ejecutar detector de versión HTTP
    res_http = ejecutar_http_version(
        client  = client,
        rhosts  = TARGET_IP,
        threads = THREADS
    )

    # Estado final
    mostrar_jobs_activos(client)
    print("\n[+] Parte A completada. Revisa la consola de msfconsole para ver resultados detallados.")
    print("    Tip: abre msfconsole y ejecuta 'jobs -l' para ver el historial.")
```

2. Ejecuta el escáner:
```bash
python lab07/msf_scanner.py
```

**Salida esperada:**
```
[+] Conexión MSFRPC establecida
[*] No hay jobs activos en Metasploit.

============================================================
[*] MÓDULO: auxiliary/scanner/portscan/tcp
[*] TARGET: 192.168.56.20  PORTS: 22,80,139,445,3306,8180  THREADS: 4
============================================================
[*] Opciones requeridas del módulo:
    RHOSTS: {'type': 'address_range', 'required': True, ...}
    PORTS: {'type': 'port_range', 'required': True, ...}

[+] Opciones configuradas: RHOSTS=192.168.56.20, PORTS=22,80,139,445,3306,8180, THREADS=4
[+] Módulo ejecutado. Job ID: 0
[*] Esperando finalización del Job ID: 0 (timeout: 20s)
    ... job 0 aún en ejecución (2s)
    ... job 0 aún en ejecución (4s)
[+] Job 0 finalizado correctamente.

============================================================
[*] MÓDULO: auxiliary/scanner/http/http_version
[*] TARGET: 192.168.56.20  THREADS: 4
============================================================
...
[+] Job 1 finalizado correctamente.

[*] No hay jobs activos en Metasploit.
[+] Parte A completada.
```

**Verificación:**
```bash
# En la terminal donde corre msfrpcd, deberías ver líneas como:
# [*] 192.168.56.20:22 - TCP OPEN
# [*] 192.168.56.20:80 - TCP OPEN
# [*] 192.168.56.20:80 Apache/2.2.8 (Ubuntu) DAV/2 ( 200 )
```

> **Nota:** Los resultados detallados de los módulos auxiliares aparecen en la **terminal de msfrpcd**, no en la salida de Python. Esto es un comportamiento normal de la API RPC: los logs del módulo van al daemon. En el Paso A4 aprenderás a capturarlos.

---

### Paso A4: Capturar Resultados via msfconsole (Verificación Complementaria)

**Objetivo:** Confirmar visualmente que los módulos ejecutados desde Python produjeron resultados reales en Metasploit.

**Instrucciones:**

1. Abre una **tercera terminal** en Kali Linux.

2. Inicia `msfconsole` en modo silencioso:
```bash
msfconsole -q
```

3. Dentro de `msfconsole`, conecta al mismo daemon RPC y verifica los resultados:
```
msf6 > db_status
# Debe mostrar: [*] Connected to msf. Connection type: postgresql.

msf6 > hosts
# Debe mostrar Metasploitable 2 si fue descubierto por el escáner

msf6 > services
# Debe mostrar los puertos abiertos detectados por portscan/tcp y http_version

msf6 > exit
```

**Salida esperada en `services`:**
```
host            port  proto  name   state  info
----            ----  -----  ----   -----  ----
192.168.56.20   22    tcp    ssh    open
192.168.56.20   80    tcp    http   open   Apache/2.2.8 (Ubuntu) DAV/2
192.168.56.20   139   tcp           open
192.168.56.20   445   tcp    smb    open
192.168.56.20   3306  tcp    mysql  open
192.168.56.20   8180  tcp    http   open
```

**Verificación:**
- Los puertos 22, 80 y al menos dos más aparecen como `open`.
- La versión HTTP muestra `Apache/2.2.8` o similar en el campo `info`.

---

### PARTE B — Enrutamiento de Tráfico a Través de TOR

---

### Paso B1: Verificar y Configurar el Daemon TOR con stem

**Objetivo:** Confirmar que el daemon TOR está activo, conectarse a él usando `stem`, y obtener información del circuito actual.

**Instrucciones:**

1. Verifica que TOR está corriendo y que el puerto de control está habilitado:
```bash
# Verificar estado del daemon
sudo systemctl status tor

# Verificar puertos: 9050 (SOCKS) y 9051 (control)
ss -tlnp | grep -E '9050|9051'
```

2. Habilitar el puerto de control de TOR (si no está ya configurado):
```bash
# Editar la configuración de TOR
sudo nano /etc/tor/torrc
```

Asegúrate de que las siguientes líneas estén presentes y sin comentar:
```
ControlPort 9051
CookieAuthentication 1
```

3. Si editaste `torrc`, reinicia el daemon:
```bash
sudo systemctl restart tor
sleep 5
sudo systemctl status tor
```

4. Crea el archivo `lab07/tor_test.py` para verificar la conexión con `stem`:

```python
#!/usr/bin/env python3
"""
Lab 07-00-01 — Parte B: Verificación de conexión TOR con stem
"""

import stem
from stem.control import Controller
from stem import Signal
import requests
import socks
import socket
import time

TOR_SOCKS_PORT   = 9050
TOR_CONTROL_PORT = 9051
TEST_URL         = 'https://check.torproject.org/api/ip'


def verificar_tor_stem() -> bool:
    """
    Conecta al puerto de control de TOR con stem y muestra
    información del estado del daemon.
    Devuelve True si la conexión fue exitosa.
    """
    print("\n[*] Verificando conexión con stem al daemon TOR...")
    try:
        with Controller.from_port(port=TOR_CONTROL_PORT) as controller:
            controller.authenticate()  # Usa autenticación por cookie
            print(f"[+] Conectado al daemon TOR via stem")
            print(f"    Versión TOR: {controller.get_version()}")
            print(f"    Estado del circuito: {controller.is_alive()}")

            # Mostrar información de los circuitos activos
            circuitos = controller.get_circuits()
            print(f"    Circuitos activos: {len(circuitos)}")
            for circ in list(circuitos)[:3]:   # Mostrar hasta 3
                ruta = ' → '.join([f"{n[1]}" for n in circ.path])
                print(f"    Circuito {circ.id}: {ruta} [{circ.status}]")

        return True
    except stem.SocketError as e:
        print(f"[-] Error de socket al conectar con TOR: {e}")
        print("    ¿Está TOR corriendo? Ejecuta: sudo systemctl start tor")
        return False
    except stem.connection.AuthenticationFailure as e:
        print(f"[-] Error de autenticación con TOR: {e}")
        print("    Verifica que CookieAuthentication 1 esté en /etc/tor/torrc")
        return False


def obtener_ip_sin_tor() -> str:
    """Obtiene la IP pública sin usar TOR (para comparación)."""
    try:
        resp = requests.get('https://api.ipify.org?format=json', timeout=10)
        return resp.json().get('ip', 'desconocida')
    except Exception:
        return 'no disponible'


def obtener_ip_con_tor() -> str:
    """
    Realiza una petición HTTP a través del proxy SOCKS5 de TOR
    y devuelve la IP observada por el servidor remoto.
    """
    proxies = {
        'http':  f'socks5h://127.0.0.1:{TOR_SOCKS_PORT}',
        'https': f'socks5h://127.0.0.1:{TOR_SOCKS_PORT}'
    }
    try:
        resp = requests.get(TEST_URL, proxies=proxies, timeout=30)
        data = resp.json()
        return data.get('IP', 'desconocida')
    except requests.exceptions.ConnectionError:
        return 'error: TOR no disponible o SOCKS5 no accesible'
    except Exception as e:
        return f'error: {e}'


# ─────────────────────────────────────────────────────────────
# Ejecución principal
# ─────────────────────────────────────────────────────────────
if __name__ == '__main__':
    # Verificar stem
    tor_ok = verificar_tor_stem()

    if not tor_ok:
        print("[-] No se puede continuar sin TOR activo.")
        exit(1)

    # Comparar IPs
    print("\n[*] Comparando IP directa vs IP a través de TOR...")
    ip_directa = obtener_ip_sin_tor()
    print(f"    IP directa (sin TOR) : {ip_directa}")

    ip_tor = obtener_ip_con_tor()
    print(f"    IP a través de TOR   : {ip_tor}")

    if ip_directa != ip_tor and 'error' not in ip_tor:
        print("[+] ¡TOR está funcionando! Las IPs son diferentes.")
    elif 'error' in ip_tor:
        print(f"[!] Error al conectar a través de TOR: {ip_tor}")
    else:
        print("[!] Las IPs son iguales. Verifica la configuración del proxy.")
```

5. Ejecuta la verificación:
```bash
python lab07/tor_test.py
```

**Salida esperada:**
```
[*] Verificando conexión con stem al daemon TOR...
[+] Conectado al daemon TOR via stem
    Versión TOR: 0.4.8.x
    Estado del circuito: True
    Circuitos activos: 5
    Circuito 1: NodeA → NodeB → NodeC [BUILT]
    Circuito 2: NodeD → NodeE → NodeF [BUILT]
    Circuito 3: NodeG → NodeH → NodeI [BUILT]

[*] Comparando IP directa vs IP a través de TOR...
    IP directa (sin TOR) : 203.0.113.45
    IP a través de TOR   : 185.220.101.x
[+] ¡TOR está funcionando! Las IPs son diferentes.
```

**Verificación:**
- La IP directa y la IP a través de TOR son **distintas**.
- `stem` reporta al menos 1 circuito en estado `BUILT`.

> **Nota importante:** Este paso requiere acceso a Internet **desde Kali Linux** (no desde la red interna de escaneo). Asegúrate de que Kali tiene salida a Internet en su adaptador principal (NAT de VirtualBox), mientras que el adaptador de red interna está aislado para el escaneo de Metasploitable 2.

---

### Paso B2: Implementar Rotación de Circuito TOR

**Objetivo:** Implementar la función de rotación de circuito TOR usando `stem` (señal `NEWNYM`) y verificar que la IP cambia tras cada rotación.

**Instrucciones:**

1. Crea el archivo `lab07/tor_rotation.py`:

```python
#!/usr/bin/env python3
"""
Lab 07-00-01 — Parte B: Rotación de circuito TOR con stem
NOTA ÉTICA: La rotación de circuito TOR en pentesting tiene límites legales.
Ver sección de discusión al final del laboratorio.
"""

from stem.control import Controller
from stem import Signal
import requests
import time
import sys

TOR_SOCKS_PORT   = 9050
TOR_CONTROL_PORT = 9051
MIN_INTERVAL_SEG = 10    # TOR recomienda mínimo 10s entre rotaciones


def rotar_circuito_tor() -> bool:
    """
    Envía la señal NEWNYM al daemon TOR para solicitar un nuevo circuito.
    Devuelve True si la señal fue enviada correctamente.
    """
    try:
        with Controller.from_port(port=TOR_CONTROL_PORT) as controller:
            controller.authenticate()
            controller.signal(Signal.NEWNYM)
            print("[+] Señal NEWNYM enviada — nuevo circuito TOR solicitado.")
            return True
    except Exception as e:
        print(f"[-] Error al rotar circuito: {e}", file=sys.stderr)
        return False


def obtener_ip_tor() -> str:
    """Obtiene la IP actual observada a través de TOR."""
    proxies = {
        'http':  f'socks5h://127.0.0.1:{TOR_SOCKS_PORT}',
        'https': f'socks5h://127.0.0.1:{TOR_SOCKS_PORT}'
    }
    try:
        resp = requests.get(
            'https://api.ipify.org?format=json',
            proxies=proxies,
            timeout=30
        )
        return resp.json().get('ip', 'desconocida')
    except Exception as e:
        return f'error: {e}'


def demostrar_rotacion(n_rotaciones: int = 3) -> list[str]:
    """
    Realiza n_rotaciones de circuito TOR y registra las IPs obtenidas.
    Respeta el intervalo mínimo recomendado entre rotaciones.
    Devuelve lista de IPs observadas.
    """
    ips_observadas = []

    print(f"\n[*] Iniciando demostración de rotación de circuito ({n_rotaciones} rotaciones)")
    print(f"[*] Intervalo mínimo entre rotaciones: {MIN_INTERVAL_SEG}s")
    print(f"{'─'*50}")

    # IP inicial antes de rotar
    ip_inicial = obtener_ip_tor()
    print(f"[0] IP inicial (circuito actual): {ip_inicial}")
    ips_observadas.append(ip_inicial)

    for i in range(1, n_rotaciones + 1):
        print(f"\n[{i}] Rotando circuito...")
        exito = rotar_circuito_tor()

        if exito:
            # Esperar el intervalo mínimo para que TOR construya el nuevo circuito
            print(f"    Esperando {MIN_INTERVAL_SEG}s para que el circuito se establezca...")
            time.sleep(MIN_INTERVAL_SEG)

            ip_nueva = obtener_ip_tor()
            print(f"[{i}] IP tras rotación {i}: {ip_nueva}")
            ips_observadas.append(ip_nueva)

            if ip_nueva != ips_observadas[-2] and 'error' not in ip_nueva:
                print(f"    ✓ IP cambió correctamente")
            else:
                print(f"    ⚠ La IP no cambió (puede ocurrir ocasionalmente con TOR)")

    print(f"\n{'─'*50}")
    print(f"[*] Resumen de IPs observadas: {ips_observadas}")
    ips_unicas = len(set(ip for ip in ips_observadas if 'error' not in ip))
    print(f"[*] IPs únicas obtenidas: {ips_unicas}/{len(ips_observadas)}")

    return ips_observadas


# ─────────────────────────────────────────────────────────────
# Discusión ética integrada
# ─────────────────────────────────────────────────────────────
DISCUSION_ETICA = """
╔══════════════════════════════════════════════════════════════╗
║         LÍMITES LEGALES DEL ANONIMATO EN PENTESTING          ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  1. TOR NO GARANTIZA ANONIMATO ABSOLUTO:                     ║
║     Los nodos de salida pueden ser monitoreados. Metadata,   ║
║     timing attacks y correlación de tráfico pueden           ║
║     deanonimizar al usuario.                                 ║
║                                                              ║
║  2. USO EN PENTESTING AUTORIZADO:                            ║
║     Enrutar tráfico de prueba por TOR puede estar            ║
║     explícitamente PROHIBIDO en el scope del contrato.       ║
║     Siempre verificar con el cliente antes de usar TOR.      ║
║                                                              ║
║  3. RESPONSABILIDAD LEGAL:                                   ║
║     Usar TOR para ocultar actividad de pentesting NO         ║
║     exime de responsabilidad legal. El contrato de           ║
║     autorización debe cubrir el método de acceso.            ║
║                                                              ║
║  4. NODOS DE SALIDA MALICIOSOS:                              ║
║     Algunos nodos de salida de TOR interceptan tráfico.      ║
║     Nunca enviar credenciales reales sin cifrado adicional.  ║
║                                                              ║
║  5. USO EN ESTE LABORATORIO:                                 ║
║     Solo verificamos conectividad TOR. NO enrutamos          ║
║     tráfico de escaneo por TOR en este laboratorio.          ║
╚══════════════════════════════════════════════════════════════╝
"""


# ─────────────────────────────────────────────────────────────
# Ejecución principal
# ─────────────────────────────────────────────────────────────
if __name__ == '__main__':
    print(DISCUSION_ETICA)

    # Demostrar rotación con 2 rotaciones (para no sobrecargar TOR)
    ips = demostrar_rotacion(n_rotaciones=2)

    print("\n[+] Parte B completada.")
```

2. Ejecuta la demostración de rotación:
```bash
python lab07/tor_rotation.py
```

**Salida esperada:**
```
╔══════════════════════════════════════════════════════════════╗
║         LÍMITES LEGALES DEL ANONIMATO EN PENTESTING          ║
...

[*] Iniciando demostración de rotación de circuito (2 rotaciones)
[*] Intervalo mínimo entre rotaciones: 10s
──────────────────────────────────────────────────
[0] IP inicial (circuito actual): 185.220.101.x

[1] Rotando circuito...
[+] Señal NEWNYM enviada — nuevo circuito TOR solicitado.
    Esperando 10s para que el circuito se establezca...
[1] IP tras rotación 1: 45.142.212.y
    ✓ IP cambió correctamente

[2] Rotando circuito...
[+] Señal NEWNYM enviada — nuevo circuito TOR solicitado.
    Esperando 10s para que el circuito se establezca...
[2] IP tras rotación 2: 176.10.99.z
    ✓ IP cambió correctamente

──────────────────────────────────────────────────
[*] Resumen de IPs observadas: ['185.220.101.x', '45.142.212.y', '176.10.99.z']
[*] IPs únicas obtenidas: 3/3

[+] Parte B completada.
```

**Verificación:**
- Al menos 2 de las 3 IPs son diferentes entre sí.
- La señal NEWNYM no lanza excepciones.

---

### PARTE C — Refactorización como Módulos Importables

---

### Paso C1: Crear `msf_module.py` como Módulo Reutilizable

**Objetivo:** Encapsular toda la funcionalidad de Metasploit en un módulo Python limpio, importable y con interfaz documentada para el toolkit final.

**Instrucciones:**

1. Crea el archivo `lab07/msf_module.py`:

```python
#!/usr/bin/env python3
"""
msf_module.py — Módulo de integración con Metasploit RPC
Lab 07-00-01 | Toolkit de Hacking Ético

Uso:
    from lab07.msf_module import MetasploitClient

    msf = MetasploitClient()
    msf.conectar()
    msf.ejecutar_portscan_tcp('192.168.56.20', '22,80,443')
    msf.ejecutar_http_version('192.168.56.20')
    msf.desconectar()

ADVERTENCIA: Solo usar en entornos con autorización escrita.
"""

from pymetasploit3.msfrpc import MsfRpcClient
import time
import sys
from typing import Optional


class MetasploitClient:
    """
    Encapsula la conexión y operaciones con Metasploit RPC.
    Proporciona métodos de alto nivel para módulos auxiliares no destructivos.
    """

    DEFAULT_CONFIG = {
        'host':     '127.0.0.1',
        'port':     55553,
        'username': 'msf',
        'password': 'Lab07Pass!',
        'ssl':      False,
        'threads':  4,
        'job_timeout': 30
    }

    def __init__(self, **kwargs):
        """
        Inicializa el cliente con configuración personalizable.
        Los kwargs sobreescriben DEFAULT_CONFIG.
        """
        self.config = {**self.DEFAULT_CONFIG, **kwargs}
        self._client: Optional[MsfRpcClient] = None

    def conectar(self) -> bool:
        """
        Establece la conexión con MSFRPC.
        Devuelve True si exitoso, False si falla.
        """
        try:
            self._client = MsfRpcClient(
                password = self.config['password'],
                username = self.config['username'],
                port     = self.config['port'],
                ssl      = self.config['ssl'],
                server   = self.config['host']
            )
            print(f"[msf_module] Conexión establecida con {self.config['host']}:{self.config['port']}")
            return True
        except Exception as e:
            print(f"[msf_module] Error de conexión: {e}", file=sys.stderr)
            return False

    def desconectar(self) -> None:
        """Cierra la conexión RPC limpiamente."""
        self._client = None
        print("[msf_module] Conexión cerrada.")

    def _requiere_conexion(self) -> None:
        """Verifica que hay una conexión activa. Lanza RuntimeError si no."""
        if self._client is None:
            raise RuntimeError("No hay conexión activa. Llama a conectar() primero.")

    def _esperar_job(self, job_id, timeout: int = None) -> bool:
        """Espera a que un job finalice. Devuelve True si terminó en tiempo."""
        if timeout is None:
            timeout = self.config['job_timeout']
        inicio = time.time()
        while time.time() - inicio < timeout:
            if str(job_id) not in self._client.jobs.list:
                return True
            time.sleep(2)
        return False

    def listar_modulos(self, termino: str, tipo: str = 'auxiliary') -> list[dict]:
        """
        Busca módulos por término y tipo.
        tipo: 'auxiliary', 'exploit', 'payload', 'post'
        """
        self._requiere_conexion()
        resultados = self._client.modules.search(termino)
        return [m for m in resultados if m.get('type') == tipo]

    def ejecutar_portscan_tcp(
        self,
        rhosts:  str,
        ports:   str = '1-1024',
        threads: int = None
    ) -> dict:
        """
        Ejecuta auxiliary/scanner/portscan/tcp.
        Devuelve {'job_id': int, 'completado': bool}.
        """
        self._requiere_conexion()
        if threads is None:
            threads = self.config['threads']

        modulo = self._client.modules.use('auxiliary', 'scanner/portscan/tcp')
        modulo['RHOSTS']  = rhosts
        modulo['PORTS']   = ports
        modulo['THREADS'] = threads

        resultado  = modulo.execute(payload='')
        job_id     = resultado.get('job_id')
        completado = self._esperar_job(job_id)

        print(f"[msf_module] portscan/tcp → Job {job_id} | "
              f"{'completado' if completado else 'timeout'}")
        return {'job_id': job_id, 'completado': completado, 'modulo': 'scanner/portscan/tcp'}

    def ejecutar_http_version(
        self,
        rhosts:  str,
        rport:   int = 80,
        threads: int = None
    ) -> dict:
        """
        Ejecuta auxiliary/scanner/http/http_version.
        Devuelve {'job_id': int, 'completado': bool}.
        """
        self._requiere_conexion()
        if threads is None:
            threads = self.config['threads']

        modulo = self._client.modules.use('auxiliary', 'scanner/http/http_version')
        modulo['RHOSTS']  = rhosts
        modulo['RPORT']   = rport
        modulo['THREADS'] = threads

        resultado  = modulo.execute(payload='')
        job_id     = resultado.get('job_id')
        completado = self._esperar_job(job_id)

        print(f"[msf_module] http/http_version → Job {job_id} | "
              f"{'completado' if completado else 'timeout'}")
        return {'job_id': job_id, 'completado': completado, 'modulo': 'scanner/http/http_version'}

    def listar_sesiones(self) -> dict:
        """Devuelve el diccionario de sesiones activas."""
        self._requiere_conexion()
        return self._client.sessions.list

    def listar_jobs(self) -> dict:
        """Devuelve el diccionario de jobs activos."""
        self._requiere_conexion()
        return self._client.jobs.list
```

---

### Paso C2: Crear `tor_module.py` como Módulo Reutilizable

**Objetivo:** Encapsular toda la funcionalidad TOR en un módulo Python limpio e importable.

**Instrucciones:**

1. Crea el archivo `lab07/tor_module.py`:

```python
#!/usr/bin/env python3
"""
tor_module.py — Módulo de enrutamiento TOR y rotación de circuito
Lab 07-00-01 | Toolkit de Hacking Ético

Uso:
    from lab07.tor_module import TorClient

    tor = TorClient()
    if tor.verificar_conexion():
        ip = tor.obtener_ip_actual()
        tor.rotar_circuito()
        session = tor.crear_sesion_requests()
        resp = session.get('https://example.com')
"""

from stem.control import Controller
from stem import Signal
import stem
import requests
import time
import sys
from typing import Optional


class TorClient:
    """
    Encapsula la interacción con el daemon TOR mediante stem.
    Proporciona métodos para verificar conexión, obtener IP y rotar circuito.
    """

    DEFAULT_CONFIG = {
        'socks_port':      9050,
        'control_port':    9051,
        'min_rotation_s':  10,    # Intervalo mínimo entre rotaciones (recomendado por TOR)
        'request_timeout': 30
    }

    def __init__(self, **kwargs):
        self.config = {**self.DEFAULT_CONFIG, **kwargs}

    def verificar_conexion(self) -> bool:
        """
        Verifica que el daemon TOR está activo y el puerto de control responde.
        Devuelve True si la conexión fue exitosa.
        """
        try:
            with Controller.from_port(port=self.config['control_port']) as ctrl:
                ctrl.authenticate()
                version = ctrl.get_version()
                print(f"[tor_module] Daemon TOR activo. Versión: {version}")
                return True
        except stem.SocketError:
            print("[tor_module] Error: no se puede conectar al puerto de control TOR.")
            print(f"             Verifica que TOR corre en el puerto {self.config['control_port']}")
            return False
        except stem.connection.AuthenticationFailure:
            print("[tor_module] Error de autenticación con TOR.")
            print("             Verifica CookieAuthentication en /etc/tor/torrc")
            return False

    def rotar_circuito(self) -> bool:
        """
        Envía señal NEWNYM para solicitar un nuevo circuito TOR.
        Espera el intervalo mínimo recomendado.
        Devuelve True si la señal fue enviada correctamente.
        """
        try:
            with Controller.from_port(port=self.config['control_port']) as ctrl:
                ctrl.authenticate()
                ctrl.signal(Signal.NEWNYM)
                print(f"[tor_module] Circuito rotado (NEWNYM). "
                      f"Esperando {self.config['min_rotation_s']}s...")
                time.sleep(self.config['min_rotation_s'])
                return True
        except Exception as e:
            print(f"[tor_module] Error al rotar circuito: {e}", file=sys.stderr)
            return False

    def crear_sesion_requests(self) -> requests.Session:
        """
        Crea y devuelve una sesión requests configurada con proxy SOCKS5 de TOR.
        Usar para peticiones HTTP que deben pasar por TOR.
        """
        session = requests.Session()
        proxy_url = f"socks5h://127.0.0.1:{self.config['socks_port']}"
        session.proxies = {
            'http':  proxy_url,
            'https': proxy_url
        }
        return session

    def obtener_ip_actual(self) -> str:
        """
        Obtiene la IP pública actual observada a través de TOR.
        Devuelve la IP como string, o 'error: ...' si falla.
        """
        session = self.crear_sesion_requests()
        try:
            resp = session.get(
                'https://api.ipify.org?format=json',
                timeout=self.config['request_timeout']
            )
            ip = resp.json().get('ip', 'desconocida')
            print(f"[tor_module] IP actual a través de TOR: {ip}")
            return ip
        except Exception as e:
            msg = f"error: {e}"
            print(f"[tor_module] {msg}", file=sys.stderr)
            return msg

    def obtener_circuitos(self) -> list:
        """
        Devuelve lista de circuitos activos del daemon TOR.
        """
        try:
            with Controller.from_port(port=self.config['control_port']) as ctrl:
                ctrl.authenticate()
                return list(ctrl.get_circuits())
        except Exception as e:
            print(f"[tor_module] Error al obtener circuitos: {e}", file=sys.stderr)
            return []
```

---

### Paso C3: Script de Integración Final

**Objetivo:** Demostrar que ambos módulos funcionan correctamente como unidades importables dentro de un script orquestador.

**Instrucciones:**

1. Crea el archivo `lab07/__init__.py` para hacer el directorio un paquete Python:
```bash
touch lab07/__init__.py
```

2. Crea el script de integración `lab07/toolkit_demo.py`:

```python
#!/usr/bin/env python3
"""
toolkit_demo.py — Demostración de integración final del toolkit
Lab 07-00-01 | Parte C

Demuestra el uso de msf_module y tor_module como componentes
importables del toolkit de hacking ético.

ADVERTENCIA: Ejecutar ÚNICAMENTE en entornos autorizados.
TARGET: Metasploitable 2 en red interna de laboratorio.
"""

import sys
import time

# Importar módulos del toolkit
from lab07.msf_module import MetasploitClient
from lab07.tor_module  import TorClient

TARGET_IP = '192.168.56.20'


def seccion(titulo: str) -> None:
    """Imprime un separador de sección formateado."""
    print(f"\n{'═'*60}")
    print(f"  {titulo}")
    print(f"{'═'*60}")


def demo_metasploit() -> bool:
    """Demuestra el uso del módulo Metasploit."""
    seccion("DEMO A: MetasploitClient — Módulos Auxiliares")

    msf = MetasploitClient(job_timeout=25)

    # Conectar
    if not msf.conectar():
        print("[-] No se pudo conectar a MSFRPC. Abortando demo MSF.")
        return False

    # Listar módulos de escaneo
    modulos = msf.listar_modulos('portscan')
    print(f"[*] Módulos portscan encontrados: {len(modulos)}")

    # Ejecutar portscan TCP
    print(f"\n[*] Ejecutando portscan TCP sobre {TARGET_IP}...")
    resultado_tcp = msf.ejecutar_portscan_tcp(
        rhosts  = TARGET_IP,
        ports   = '21,22,23,25,80,139,445,3306,8180',
        threads = 4
    )
    print(f"[*] Resultado portscan: {resultado_tcp}")

    time.sleep(2)

    # Ejecutar http_version
    print(f"\n[*] Ejecutando http_version sobre {TARGET_IP}:80...")
    resultado_http = msf.ejecutar_http_version(
        rhosts  = TARGET_IP,
        rport   = 80,
        threads = 4
    )
    print(f"[*] Resultado http_version: {resultado_http}")

    # Verificar sesiones (no esperamos ninguna con módulos auxiliares)
    sesiones = msf.listar_sesiones()
    print(f"\n[*] Sesiones activas: {len(sesiones)}")

    msf.desconectar()
    return True


def demo_tor() -> bool:
    """Demuestra el uso del módulo TOR."""
    seccion("DEMO B: TorClient — Enrutamiento y Rotación")

    tor = TorClient()

    # Verificar conexión
    if not tor.verificar_conexion():
        print("[-] TOR no disponible. Abortando demo TOR.")
        return False

    # Obtener IP inicial
    ip_inicial = tor.obtener_ip_actual()

    # Mostrar circuitos
    circuitos = tor.obtener_circuitos()
    print(f"[*] Circuitos TOR activos: {len(circuitos)}")

    # Rotar circuito
    print("[*] Rotando circuito TOR...")
    tor.rotar_circuito()

    # Verificar nueva IP
    ip_nueva = tor.obtener_ip_actual()

    # Resumen
    print(f"\n[*] IP antes de rotación : {ip_inicial}")
    print(f"[*] IP después de rotación: {ip_nueva}")
    cambio = ip_inicial != ip_nueva and 'error' not in ip_nueva
    print(f"[*] ¿IP cambió? {'Sí ✓' if cambio else 'No (puede ser normal)'}")

    return True


def main():
    print("\n" + "█"*60)
    print("  TOOLKIT DEMO — Lab 07-00-01")
    print("  Integración Metasploit + TOR")
    print("█"*60)

    resultados = {}

    # Demo Metasploit
    resultados['metasploit'] = demo_metasploit()

    # Demo TOR
    resultados['tor'] = demo_tor()

    # Resumen final
    seccion("RESUMEN FINAL")
    for componente, exito in resultados.items():
        estado = "✓ OK" if exito else "✗ FALLO"
        print(f"  {componente:15s}: {estado}")

    exito_total = all(resultados.values())
    print(f"\n{'[+] Toolkit demo completado exitosamente.' if exito_total else '[!] Algunos componentes fallaron. Revisar logs.'}")
    return 0 if exito_total else 1


if __name__ == '__main__':
    sys.exit(main())
```

3. Ejecuta el script de integración final:
```bash
cd ~/eth-toolkit
python -m lab07.toolkit_demo
```

**Salida esperada:**
```
████████████████████████████████████████████████████████████
  TOOLKIT DEMO — Lab 07-00-01
  Integración Metasploit + TOR
████████████████████████████████████████████████████████████

════════════════════════════════════════════════════════════
  DEMO A: MetasploitClient — Módulos Auxiliares
════════════════════════════════════════════════════════════
[msf_module] Conexión establecida con 127.0.0.1:55553
[*] Módulos portscan encontrados: 5
[*] Ejecutando portscan TCP sobre 192.168.56.20...
[msf_module] portscan/tcp → Job 2 | completado
[*] Resultado portscan: {'job_id': 2, 'completado': True, 'modulo': 'scanner/portscan/tcp'}
[*] Ejecutando http_version sobre 192.168.56.20:80...
[msf_module] http/http_version → Job 3 | completado
[*] Resultado http_version: {'job_id': 3, 'completado': True, 'modulo': 'scanner/http/http_version'}
[*] Sesiones activas: 0
[msf_module] Conexión cerrada.

════════════════════════════════════════════════════════════
  DEMO B: TorClient — Enrutamiento y Rotación
════════════════════════════════════════════════════════════
[tor_module] Daemon TOR activo. Versión: 0.4.8.x
[tor_module] IP actual a través de TOR: 185.220.101.x
[*] Circuitos TOR activos: 6
[*] Rotando circuito TOR...
[tor_module] Circuito rotado (NEWNYM). Esperando 10s...
[tor_module] IP actual a través de TOR: 45.142.212.y
[*] IP antes de rotación : 185.220.101.x
[*] IP después de rotación: 45.142.212.y
[*] ¿IP cambió? Sí ✓

════════════════════════════════════════════════════════════
  RESUMEN FINAL
════════════════════════════════════════════════════════════
  metasploit    : ✓ OK
  tor           : ✓ OK

[+] Toolkit demo completado exitosamente.
```

**Verificación:**
- Ambos componentes reportan `✓ OK` en el resumen final.
- Los Jobs de Metasploit muestran estado `completado`.
- La IP de TOR cambia tras la rotación.

---

## 7. Validación y Pruebas

Ejecuta los siguientes comandos de validación para confirmar que el laboratorio está completo:

```bash
# 1. Verificar que todos los archivos del módulo existen
ls -la ~/eth-toolkit/lab07/
# Debe mostrar: __init__.py, msf_module.py, tor_module.py,
#               msf_test.py, msf_scanner.py, tor_test.py,
#               tor_rotation.py, toolkit_demo.py

# 2. Verificar importabilidad de los módulos
cd ~/eth-toolkit
python -c "from lab07.msf_module import MetasploitClient; print('[OK] msf_module importable')"
python -c "from lab07.tor_module import TorClient; print('[OK] tor_module importable')"

# 3. Verificar que msfrpcd sigue activo
ss -tlnp | grep 55553 && echo "[OK] msfrpcd activo"

# 4. Verificar que TOR sigue activo
sudo systemctl is-active tor && echo "[OK] TOR daemon activo"

# 5. Verificar resultados en Metasploit (en msfconsole)
# msfconsole -q -x "services; exit"
# Debe mostrar los puertos descubiertos en 192.168.56.20

# 6. Ejecutar suite de validación completa
python -m lab07.toolkit_demo
# Debe mostrar: metasploit: ✓ OK  y  tor: ✓ OK
```

**Criterios de éxito:**

| Criterio | Verificación |
|----------|-------------|
| `msf_module.py` importable sin errores | `python -c "from lab07.msf_module import MetasploitClient"` |
| `tor_module.py` importable sin errores | `python -c "from lab07.tor_module import TorClient"` |
| portscan/tcp ejecutado y completado | Job devuelve `completado: True` |
| http_version ejecutado y completado | Job devuelve `completado: True` |
| TOR conecta y obtiene IP externa | IP diferente a IP directa de Kali |
| Rotación de circuito produce nueva IP | Al menos 2 de 3 IPs son distintas |
| Toolkit demo finaliza con `✓ OK` en ambos | Salida del resumen final |

---

## 8. Solución de Problemas

### Problema 1: `ConnectionRefusedError` al conectar con `MsfRpcClient`

**Síntomas:**
```
[-] Error de conexión MSFRPC: [Errno 111] Connection refused
    ¿Está msfrpcd ejecutándose? Verifica con: ss -tlnp | grep 55553
```
El script Python no puede establecer conexión con el daemon RPC de Metasploit.

**Causa:**
`msfrpcd` no está corriendo, se inició con un puerto diferente, o fue terminado accidentalmente. También puede ocurrir si se inició con `ssl=True` pero el cliente usa `ssl=False` (o viceversa).

**Solución:**
```bash
# 1. Verificar si msfrpcd está corriendo
ss -tlnp | grep 55553
pgrep -a ruby | grep msfrpcd

# 2. Si no está corriendo, iniciarlo nuevamente
msfrpcd -P Lab07Pass! -U msf -p 55553 -S

# 3. Esperar 15 segundos y verificar nuevamente
sleep 15 && ss -tlnp | grep 55553

# 4. Si el problema persiste, verificar que el puerto no está bloqueado por firewall
sudo ufw status
# Si UFW está activo: sudo ufw allow 55553/tcp

# 5. Verificar consistencia SSL: si msfrpcd se inició con -S, el cliente debe tener ssl=False
# Si msfrpcd se inició SIN -S, el cliente debe tener ssl=True
```

---

### Problema 2: `stem.SocketError` o `AuthenticationFailure` al conectar con TOR

**Síntomas:**
```
[tor_module] Error: no se puede conectar al puerto de control TOR.
             Verifica que TOR corre en el puerto 9051
```
O bien:
```
[tor_module] Error de autenticación con TOR.
             Verifica CookieAuthentication en /etc/tor/torrc
```

**Causa:**
El puerto de control TOR (9051) no está habilitado en `torrc`, el daemon TOR no está corriendo, o el usuario no tiene permisos para leer el archivo de cookie de autenticación (`/run/tor/control.authcookie`).

**Solución:**
```bash
# 1. Verificar estado del daemon TOR
sudo systemctl status tor

# 2. Verificar configuración de torrc
grep -E 'ControlPort|CookieAuthentication' /etc/tor/torrc
# Debe mostrar:
# ControlPort 9051
# CookieAuthentication 1

# 3. Si las líneas no existen o están comentadas, editarlas
sudo nano /etc/tor/torrc
# Descomentar o agregar:
# ControlPort 9051
# CookieAuthentication 1

# 4. Reiniciar TOR y verificar puertos
sudo systemctl restart tor
sleep 5
ss -tlnp | grep -E '9050|9051'

# 5. Verificar permisos del archivo de cookie
ls -la /run/tor/control.authcookie
# Si el usuario actual no tiene acceso, agregar al grupo debian-tor:
sudo usermod -aG debian-tor $USER
# Luego cerrar sesión y volver a entrar (o usar: newgrp debian-tor)

# 6. Probar conexión manual con stem
python3 -c "
from stem.control import Controller
with Controller.from_port(port=9051) as c:
    c.authenticate()
    print('OK:', c.get_version())
"
```

---

## 9. Limpieza del Entorno

Ejecuta los siguientes pasos al finalizar el laboratorio:

```bash
# ─────────────────────────────────────────────────────────────
# 1. Detener msfrpcd
# ─────────────────────────────────────────────────────────────
# Presiona Ctrl+C en la terminal donde corre msfrpcd, o:
pkill -f msfrpcd
echo "[+] msfrpcd detenido"

# ─────────────────────────────────────────────────────────────
# 2. Detener el daemon TOR (opcional; se puede dejar activo)
# ─────────────────────────────────────────────────────────────
sudo systemctl stop tor
echo "[+] TOR daemon detenido"

# ─────────────────────────────────────────────────────────────
# 3. Restaurar snapshot de Metasploitable 2 (desde host)
# ─────────────────────────────────────────────────────────────
# En VirtualBox (desde terminal del host, con la VM apagada):
VBoxManage controlvm "Metasploitable2" poweroff 2>/dev/null
sleep 3
VBoxManage snapshot "Metasploitable2" restore "Pre-Lab07"
echo "[+] Snapshot Pre-Lab07 restaurado en Metasploitable 2"

# ─────────────────────────────────────────────────────────────
# 4. Limpiar base de datos de Metasploit (opcional)
# ─────────────────────────────────────────────────────────────
# Si deseas limpiar los hosts/servicios registrados en este lab:
# msfconsole -q -x "db_nmap -h; hosts -d 192.168.56.20; exit"

# ─────────────────────────────────────────────────────────────
# 5. Desactivar entorno virtual
# ─────────────────────────────────────────────────────────────
deactivate
echo "[+] Entorno virtual desactivado"

# ─────────────────────────────────────────────────────────────
# 6. Verificar que no quedan procesos residuales
# ─────────────────────────────────────────────────────────────
pgrep -a ruby | grep msfrpcd && echo "[!] msfrpcd aún activo" || echo "[+] msfrpcd no activo"
ss -tlnp | grep -E '55553|9051' && echo "[!] Puertos aún abiertos" || echo "[+] Puertos liberados"
```

> **Nota:** Los archivos del módulo (`msf_module.py`, `tor_module.py`) deben **conservarse** en el repositorio del toolkit para los labs siguientes. No eliminar el directorio `lab07/`.

---

## 10. Resumen

### Lo que construiste en este laboratorio

En este laboratorio de nivel **Crear** completaste tres partes integradas:

| Parte | Logro |
|-------|-------|
| **A — Metasploit RPC** | Iniciaste `msfrpcd`, conectaste Python con `pymetasploit3`, automatizaste `portscan/tcp` y `http_version` contra Metasploitable 2, y monitorizaste jobs con espera inteligente. |
| **B — TOR** | Verificaste el daemon TOR con `stem`, comparaste IPs directa vs. TOR, implementaste rotación de circuito con `Signal.NEWNYM`, y discutiste los límites legales del anonimato. |
| **C — Módulos reutilizables** | Refactorizaste ambas funcionalidades en `MetasploitClient` y `TorClient`, clases importables con interfaces limpias, manejo de errores y documentación para el toolkit final. |

### Conceptos clave consolidados

- **`msfrpcd`** expone la API de Metasploit mediante MessagePack sobre HTTP; `pymetasploit3` abstrae esa comunicación en objetos Python.
- Los módulos auxiliares de Metasploit son **no destructivos** por diseño; son herramientas de reconocimiento, no de explotación.
- **`stem`** permite controlar el daemon TOR programáticamente: verificar estado, listar circuitos y enviar señales como `NEWNYM`.
- La rotación de circuito TOR **no garantiza anonimato completo**; su uso en pentesting debe estar explícitamente autorizado en el scope del contrato.
- La refactorización en módulos importables es fundamental para el **toolkit final**: permite composición, pruebas unitarias y mantenimiento independiente de cada componente.

### Recursos Adicionales

- [Documentación oficial de Metasploit Framework — Rapid7](https://docs.metasploit.com/)
- [Repositorio de pymetasploit3 en GitHub](https://github.com/DanMcInerney/pymetasploit3)
- [Documentación de stem — Control de TOR desde Python](https://stem.torproject.org/)
- [Referencia de la API RPC de Metasploit](https://metasploit.help.rapid7.com/docs/rpc-api)
- [Proyecto TOR — Documentación técnica](https://www.torproject.org/docs/)
- [PySocks — Repositorio oficial](https://github.com/Anorov/PySocks)

---
*Lab 07-00-01 | Curso de Hacking Ético con Python | Todos los ejercicios deben ejecutarse exclusivamente en entornos propios o máquinas virtuales con autorización explícita documentada.*
