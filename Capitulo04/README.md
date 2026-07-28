# Práctica 4 — Herramienta de Banner Grabbing en Python

## 1. Metadatos

| Campo            | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 46 minutos                                   |
| **Complejidad**  | Media                                        |
| **Nivel Bloom**  | Crear (*Create*)                             |
| **Módulo**       | 4 — Fingerprinting de Servicios              |
| **Herramientas** | Python 3.10+, socket, requests, re, concurrent.futures, json, Metasploitable 2 |

---

## 2. Descripción General

En esta práctica construirás desde cero una herramienta de **fingerprinting activo** basada en sockets Python que conecta a puertos TCP comunes de la máquina Metasploitable 2, captura los banners de respuesta y los clasifica automáticamente mediante expresiones regulares comparadas contra un diccionario de firmas conocidas. Al finalizar, la herramienta generará un reporte estructurado en JSON que indica, por cada puerto, el servicio detectado, la versión identificada, el banner crudo y una clasificación básica de riesgo. Todo el escaneo se realiza exclusivamente sobre la red NAT interna del laboratorio; ningún paquete sale hacia Internet.

---

## 3. Objetivos de Aprendizaje

- [ ] Implementar banner grabbing sobre puertos TCP comunes (21, 22, 25, 80, 443, 3306, 8080) usando el módulo `socket` de Python.
- [ ] Extraer y analizar cabeceras HTTP con `requests` para complementar el fingerprinting de servicios web.
- [ ] Clasificar automáticamente los banners capturados mediante expresiones regulares y un diccionario de firmas de servicios.
- [ ] Generar un reporte estructurado en JSON con servicio, versión detectada, puerto y nivel de riesgo básico.
- [ ] Aplicar escaneo paralelo de puertos con `concurrent.futures` respetando principios de uso responsable de recursos.

---

## 4. Prerrequisitos

### Conocimiento previo
- Haber completado el **Lab 01-00-01** (fundamentos de sockets Python).
- Comprensión de protocolos TCP/IP, handshake de tres vías y puertos bien conocidos.
- Familiaridad con expresiones regulares básicas en Python (`re.search`, grupos de captura).
- Conocimiento del concepto de fingerprinting activo vs. pasivo (Lección 4.1).

### Acceso y autorizaciones
- Metasploitable 2 desplegada y accesible en la red NAT interna de VirtualBox/VMware.
- **Formulario de autorización firmado** por el instructor para escanear la VM objetivo (requerido antes de ejecutar cualquier escaneo activo).
- Kali Linux (o la VM de trabajo) en la misma red NAT interna que Metasploitable 2.
- Acceso a terminal con Python 3.10+ y pip disponibles.

---

## 5. Entorno de Laboratorio

### Hardware mínimo requerido

| Componente        | Mínimo requerido                                    |
|-------------------|-----------------------------------------------------|
| CPU               | 64 bits con VT-x/AMD-V habilitado, 4 núcleos        |
| RAM               | 8 GB (16 GB recomendado para VMs simultáneas)       |
| Disco             | 60 GB libres para snapshots e imágenes              |
| Red               | Adaptador compatible con modo promiscuo             |
| Pantalla          | 1280×768 mínimo (múltiples terminales)              |

### Software requerido

| Software                  | Versión mínima  | Rol en el laboratorio                        |
|---------------------------|-----------------|----------------------------------------------|
| Python                    | 3.10+           | Lenguaje principal de la herramienta         |
| requests                  | 2.31+           | Extracción de cabeceras HTTP                 |
| Metasploitable 2          | —               | Objetivo de escaneo (VM aislada)             |
| VirtualBox / VMware       | 7.0+ / 17+      | Virtualización y red NAT interna             |
| Kali Linux                | 2024.1+         | VM de trabajo del estudiante                 |

### Preparación del entorno

> ⚠️ **IMPORTANTE — Aislamiento de red:** Antes de comenzar, verifica que la red NAT interna de VirtualBox/VMware **no tenga acceso a Internet**. Metasploitable 2 no debe poder enrutar tráfico fuera de la red del laboratorio.

#### Paso 0-A: Crear snapshot de Metasploitable 2

Antes de cualquier escaneo activo, crea un snapshot para poder restaurar el estado limpio:

```bash
# En VirtualBox (desde la VM apagada o en modo guardado)
VBoxManage snapshot "Metasploitable2" take "pre-lab04" \
  --description "Estado limpio antes del Lab 04-00-01"

# Verificar que el snapshot fue creado
VBoxManage snapshot "Metasploitable2" list
```

> Si usas VMware Workstation: **VM → Snapshot → Take Snapshot → "pre-lab04"**

#### Paso 0-B: Verificar conectividad con Metasploitable 2

```bash
# Desde la VM Kali Linux, identificar la IP de Metasploitable 2
# (normalmente en el rango 192.168.x.x de la red NAT interna)
ping -c 3 192.168.56.101   # Ajusta la IP según tu configuración

# Verificar que los puertos objetivo están accesibles
nc -zv 192.168.56.101 21 22 25 80 3306 8080
```

#### Paso 0-C: Preparar el entorno virtual de Python

```bash
# Crear directorio del proyecto
mkdir -p ~/labs/lab04-banner-grabbing
cd ~/labs/lab04-banner-grabbing

# Crear y activar entorno virtual
python3 -m venv venv
source venv/bin/activate

# Instalar dependencias
pip install requests==2.31.0

# Verificar instalación
python3 -c "import socket, requests, re, concurrent.futures, json; print('Dependencias OK')"
```

---

## 6. Instrucciones Paso a Paso

---

### Paso 1: Crear la estructura del proyecto y el módulo de configuración

**Objetivo:** Establecer la arquitectura modular de la herramienta y definir los parámetros de escaneo de forma centralizada.

#### Instrucciones

1. Desde la terminal en `~/labs/lab04-banner-grabbing` (con el entorno virtual activo), crea la estructura de archivos:

```bash
mkdir -p output
touch banner_grabber.py signatures.py report.py
ls -la
```

2. Abre `banner_grabber.py` con tu editor preferido (`nano`, `vim` o VS Code) y agrega el bloque de configuración inicial:

```python
#!/usr/bin/env python3
"""
banner_grabber.py
Herramienta de banner grabbing y fingerprinting de servicios.
Lab 04-00-01 — Uso exclusivo sobre entornos autorizados.
"""

import socket
import re
import json
import concurrent.futures
from datetime import datetime
from typing import Optional

import requests
from requests.exceptions import RequestException

# ─────────────────────────────────────────────
#  CONFIGURACIÓN CENTRAL
# ─────────────────────────────────────────────

# MODIFICA ESTA IP según la dirección de tu Metasploitable 2
TARGET_HOST = "192.168.56.101"

# Puertos a escanear con su nombre de servicio esperado
TARGET_PORTS = {
    21:   "FTP",
    22:   "SSH",
    25:   "SMTP",
    80:   "HTTP",
    443:  "HTTPS",
    3306: "MySQL",
    8080: "HTTP-alt",
}

# Parámetros de conexión
SOCKET_TIMEOUT   = 4.0   # segundos
HTTP_TIMEOUT     = 5.0   # segundos para requests
MAX_BANNER_BYTES = 2048  # bytes máximos a leer del banner
MAX_WORKERS      = 4     # hilos paralelos para concurrent.futures

# Niveles de riesgo básico
RISK_LEVELS = {
    "CRITICAL": "Versión con vulnerabilidades críticas conocidas (CVE publicados)",
    "HIGH":     "Versión desactualizada con vulnerabilidades de alto impacto",
    "MEDIUM":   "Versión con posibles vulnerabilidades o configuración débil",
    "LOW":      "Versión reciente o sin CVEs críticos conocidos",
    "INFO":     "Servicio detectado pero sin clasificación de riesgo disponible",
    "UNKNOWN":  "No se pudo identificar la versión del servicio",
}
```

3. Guarda el archivo.

**Salida esperada:**

```
banner_grabber.py  output/  report.py  signatures.py
```

**Verificación:**

```bash
python3 -c "import banner_grabber" 2>&1 | head -5
# No debe mostrar errores de importación en este punto
```

---

### Paso 2: Construir el diccionario de firmas de servicios

**Objetivo:** Crear el módulo `signatures.py` con un diccionario de patrones regex que permita clasificar los banners capturados según servicio, versión y nivel de riesgo.

#### Instrucciones

1. Abre `signatures.py` y escribe el siguiente contenido completo:

```python
"""
signatures.py
Diccionario de firmas de servicios para clasificación de banners.
Cada firma contiene: patrón regex, nombre del servicio, nivel de riesgo
y una nota explicativa del riesgo.

NOTA: Las versiones marcadas como CRITICAL/HIGH corresponden a versiones
presentes en Metasploitable 2, usadas aquí con fines educativos.
"""

# Estructura de cada firma:
# {
#   "service":  nombre legible del servicio,
#   "pattern":  expresión regular para detectar la versión,
#   "risk":     nivel de riesgo (CRITICAL, HIGH, MEDIUM, LOW, INFO, UNKNOWN),
#   "note":     descripción breve del riesgo o característica,
#   "version_group": índice del grupo de captura que contiene la versión (0 = match completo)
# }

SIGNATURES = [

    # ── FTP ──────────────────────────────────────────────────────────────
    {
        "service": "ProFTPD",
        "pattern": r"ProFTPD\s+([\d\.]+)",
        "risk": "CRITICAL",
        "note": "ProFTPD 1.3.3c presente en Metasploitable 2 — backdoor mod_copy (CVE-2015-3306)",
        "version_group": 1,
    },
    {
        "service": "vsftpd",
        "pattern": r"vsftpd\s+([\d\.]+)",
        "risk": "CRITICAL",
        "note": "vsftpd 2.3.4 presente en Metasploitable 2 — backdoor conocido en puerto 6200",
        "version_group": 1,
    },
    {
        "service": "FTP-Generic",
        "pattern": r"^220[- ](.{1,60})",
        "risk": "INFO",
        "note": "Servidor FTP genérico; versión no identificada por firma específica",
        "version_group": 1,
    },

    # ── SSH ──────────────────────────────────────────────────────────────
    {
        "service": "OpenSSH",
        "pattern": r"SSH-[\d\.]+-OpenSSH_([\d\.p]+)",
        "risk": "HIGH",
        "note": "OpenSSH versión antigua detectada; verificar CVEs específicos de la versión",
        "version_group": 1,
    },
    {
        "service": "SSH-Generic",
        "pattern": r"^SSH-([\d\.]+)-(.+)",
        "risk": "INFO",
        "note": "Implementación SSH no identificada por firma específica",
        "version_group": 2,
    },

    # ── SMTP ─────────────────────────────────────────────────────────────
    {
        "service": "Postfix",
        "pattern": r"220.*Postfix",
        "risk": "MEDIUM",
        "note": "Servidor Postfix; verificar configuración de relay abierto",
        "version_group": 0,
    },
    {
        "service": "Sendmail",
        "pattern": r"220.*Sendmail\s+([\d\.]+)",
        "risk": "HIGH",
        "note": "Sendmail con versión expuesta; historial de vulnerabilidades conocido",
        "version_group": 1,
    },
    {
        "service": "SMTP-Generic",
        "pattern": r"^220[- ](.{1,80})",
        "risk": "INFO",
        "note": "Servidor SMTP genérico detectado",
        "version_group": 1,
    },

    # ── HTTP / Apache ─────────────────────────────────────────────────────
    {
        "service": "Apache",
        "pattern": r"Apache/([\d\.]+)",
        "risk": "HIGH",
        "note": "Apache versión expuesta en cabecera Server; versiones < 2.4.50 tienen CVEs críticos",
        "version_group": 1,
    },
    {
        "service": "Nginx",
        "pattern": r"nginx/([\d\.]+)",
        "risk": "MEDIUM",
        "note": "Nginx con versión expuesta en cabecera Server",
        "version_group": 1,
    },
    {
        "service": "IIS",
        "pattern": r"Microsoft-IIS/([\d\.]+)",
        "risk": "MEDIUM",
        "note": "Microsoft IIS con versión expuesta",
        "version_group": 1,
    },
    {
        "service": "PHP",
        "pattern": r"PHP/([\d\.]+)",
        "risk": "HIGH",
        "note": "Versión de PHP expuesta en cabecera X-Powered-By; PHP < 7.4 con múltiples CVEs",
        "version_group": 1,
    },

    # ── MySQL ─────────────────────────────────────────────────────────────
    {
        "service": "MySQL",
        "pattern": r"([\d\.]+)-.*MySQL",
        "risk": "CRITICAL",
        "note": "MySQL expuesto en puerto 3306; en Metasploitable 2 acepta conexiones sin contraseña",
        "version_group": 1,
    },
    {
        "service": "MySQL-Alt",
        "pattern": r"\x00\x00\x00\x0a([\d\.]+)",
        "risk": "CRITICAL",
        "note": "MySQL detectado por handshake binario; versión en banner de bienvenida",
        "version_group": 1,
    },

    # ── Genérico ──────────────────────────────────────────────────────────
    {
        "service": "Unknown",
        "pattern": r".+",
        "risk": "UNKNOWN",
        "note": "Banner capturado pero no coincide con ninguna firma conocida",
        "version_group": 0,
    },
]


def clasificar_banner(banner: str) -> dict:
    """
    Compara un banner contra el diccionario de firmas y devuelve
    la primera coincidencia encontrada con su clasificación.

    Args:
        banner: Cadena de texto del banner capturado.

    Returns:
        Diccionario con keys: service, version, risk, note.
    """
    if not banner or banner.startswith("["):
        return {
            "service": "N/A",
            "version": "N/A",
            "risk": "UNKNOWN",
            "note": "No se pudo capturar el banner o el puerto está cerrado",
        }

    for firma in SIGNATURES:
        match = re.search(firma["pattern"], banner, re.IGNORECASE | re.DOTALL)
        if match:
            grupo = firma["version_group"]
            try:
                version = match.group(grupo) if grupo > 0 else match.group(0)
                version = version.strip()[:60]  # Limitar longitud
            except IndexError:
                version = "detectada (versión no extraída)"

            return {
                "service": firma["service"],
                "version": version,
                "risk":    firma["risk"],
                "note":    firma["note"],
            }

    # Fallback si ninguna firma coincide (no debería llegar aquí
    # dado que el último patrón es ".+", pero por seguridad:)
    return {
        "service": "Unknown",
        "version": "N/A",
        "risk": "UNKNOWN",
        "note": "Banner capturado sin clasificación disponible",
    }
```

2. Guarda el archivo.

**Verificación rápida del módulo de firmas:**

```bash
python3 -c "
from signatures import clasificar_banner
test_banners = [
    '220 ProFTPD 1.3.3c Server (Metasploitable)',
    'SSH-2.0-OpenSSH_4.7p1 Debian-8ubuntu1',
    'Apache/2.2.8 (Ubuntu) DAV/2',
]
for b in test_banners:
    r = clasificar_banner(b)
    print(f'Banner: {b[:40]!r}')
    print(f'  -> Servicio: {r[\"service\"]} | Versión: {r[\"version\"]} | Riesgo: {r[\"risk\"]}')
    print()
"
```

**Salida esperada:**

```
Banner: '220 ProFTPD 1.3.3c Server (Metasploitable)'
  -> Servicio: ProFTPD | Versión: 1.3.3c | Riesgo: CRITICAL

Banner: 'SSH-2.0-OpenSSH_4.7p1 Debian-8ubuntu1'
  -> Servicio: OpenSSH | Versión: 4.7p1 | Riesgo: HIGH

Banner: 'Apache/2.2.8 (Ubuntu) DAV/2'
  -> Servicio: Apache | Versión: 2.2.8 | Riesgo: HIGH
```

---

### Paso 3: Implementar las funciones de captura de banners

**Objetivo:** Construir las funciones principales de captura: una para banners TCP genéricos y otra especializada para servicios HTTP/HTTPS usando `requests`.

#### Instrucciones

1. Abre `banner_grabber.py` y **agrega** (al final del archivo) las siguientes funciones:

```python
# ─────────────────────────────────────────────
#  FUNCIONES DE CAPTURA DE BANNERS
# ─────────────────────────────────────────────

def capturar_banner_tcp(host: str, puerto: int,
                        timeout: float = SOCKET_TIMEOUT) -> str:
    """
    Conecta a host:puerto mediante TCP y captura el banner de bienvenida
    que el servicio envía espontáneamente al establecer la conexión.

    Para servicios que no envían banner espontáneo, envía una sonda
    específica según el puerto para provocar una respuesta.

    Args:
        host:    Dirección IP o hostname del objetivo.
        puerto:  Puerto TCP destino.
        timeout: Tiempo máximo de espera en segundos.

    Returns:
        Banner capturado como string, o mensaje de error entre corchetes.
    """
    # Sondas específicas por puerto para servicios que no envían
    # banner espontáneo al conectar
    SONDAS = {
        80:   b"HEAD / HTTP/1.0\r\nHost: " + host.encode() + b"\r\n\r\n",
        443:  b"HEAD / HTTP/1.0\r\nHost: " + host.encode() + b"\r\n\r\n",
        8080: b"HEAD / HTTP/1.0\r\nHost: " + host.encode() + b"\r\n\r\n",
        3306: None,  # MySQL envía banner binario espontáneamente
        25:   None,  # SMTP envía banner espontáneo
        21:   None,  # FTP envía banner espontáneo
        22:   None,  # SSH envía banner espontáneo
    }

    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.settimeout(timeout)
            s.connect((host, puerto))

            # Si hay sonda definida para este puerto, la enviamos
            sonda = SONDAS.get(puerto)
            if sonda is not None:
                s.sendall(sonda)

            # Recibir respuesta
            datos = s.recv(MAX_BANNER_BYTES)
            banner = datos.decode("utf-8", errors="replace").strip()
            return banner

    except socket.timeout:
        return "[Timeout — sin respuesta]"
    except ConnectionRefusedError:
        return "[Puerto cerrado — conexión rechazada]"
    except OSError as e:
        return f"[Error de red: {e}]"
    except Exception as e:
        return f"[Error inesperado: {e}]"


def capturar_cabeceras_http(host: str, puerto: int,
                             timeout: float = HTTP_TIMEOUT) -> dict:
    """
    Realiza una petición HEAD al servicio HTTP/HTTPS y extrae
    las cabeceras relevantes para fingerprinting.

    Usa requests en lugar de sockets raw para manejar correctamente
    TLS y redirecciones.

    Args:
        host:   IP o hostname del servidor web.
        puerto: Puerto del servicio (80, 443, 8080, etc.).
        timeout: Tiempo máximo de espera.

    Returns:
        Diccionario con las cabeceras capturadas y el banner construido.
    """
    esquema = "https" if puerto == 443 else "http"
    url = f"{esquema}://{host}:{puerto}/"

    cabeceras_interes = [
        "Server", "X-Powered-By", "X-Generator",
        "X-AspNet-Version", "X-Runtime", "Via",
        "Set-Cookie",  # Puede revelar frameworks (e.g., PHPSESSID)
    ]

    resultado = {
        "url":     url,
        "status":  None,
        "headers": {},
        "banner":  "",
    }

    try:
        respuesta = requests.head(
            url,
            timeout=timeout,
            verify=False,          # Metasploitable 2 usa cert autofirmado
            allow_redirects=True,
            headers={"User-Agent": "BannerGrabber/1.0 (Lab04 Educational)"},
        )
        resultado["status"] = respuesta.status_code

        for cabecera in cabeceras_interes:
            valor = respuesta.headers.get(cabecera)
            if valor:
                resultado["headers"][cabecera] = valor

        # Construir banner sintético para clasificación
        partes_banner = []
        if "Server" in resultado["headers"]:
            partes_banner.append(resultado["headers"]["Server"])
        if "X-Powered-By" in resultado["headers"]:
            partes_banner.append(resultado["headers"]["X-Powered-By"])

        resultado["banner"] = " | ".join(partes_banner) if partes_banner \
            else f"HTTP {respuesta.status_code} (sin cabecera Server)"

    except requests.exceptions.SSLError:
        resultado["banner"] = "[SSL Error — certificado inválido]"
    except RequestException as e:
        resultado["banner"] = f"[HTTP Error: {e}]"

    return resultado
```

2. Guarda el archivo.

**Verificación de las funciones de captura** (asegúrate de que Metasploitable 2 esté encendida):

```bash
python3 -c "
import sys
sys.path.insert(0, '.')
from banner_grabber import capturar_banner_tcp, capturar_cabeceras_http, TARGET_HOST

# Prueba FTP (puerto 21)
print('=== Prueba FTP (puerto 21) ===')
banner = capturar_banner_tcp(TARGET_HOST, 21)
print(f'Banner: {banner[:100]}')

# Prueba SSH (puerto 22)
print()
print('=== Prueba SSH (puerto 22) ===')
banner = capturar_banner_tcp(TARGET_HOST, 22)
print(f'Banner: {banner[:100]}')

# Prueba HTTP (puerto 80) con requests
print()
print('=== Prueba HTTP (puerto 80) ===')
resultado = capturar_cabeceras_http(TARGET_HOST, 80)
print(f'Status: {resultado[\"status\"]}')
print(f'Banner: {resultado[\"banner\"]}')
print(f'Cabeceras: {resultado[\"headers\"]}')
"
```

**Salida esperada** (los valores exactos dependen de tu Metasploitable 2):

```
=== Prueba FTP (puerto 21) ===
Banner: 220 ProFTPD 1.3.3c Server (ProFTPD Default Installation) [192.168.56.101]

=== Prueba SSH (puerto 22) ===
Banner: SSH-2.0-OpenSSH_4.7p1 Debian-8ubuntu1

=== Prueba HTTP (puerto 80) ===
Status: 200
Banner: Apache/2.2.8 (Ubuntu) DAV/2
Cabeceras: {'Server': 'Apache/2.2.8 (Ubuntu) DAV/2', 'X-Powered-By': 'PHP/5.2.4-2ubuntu5.10'}
```

---

### Paso 4: Implementar el escáner paralelo de puertos

**Objetivo:** Integrar el escaneo de todos los puertos objetivo usando `concurrent.futures.ThreadPoolExecutor` para ejecutar las conexiones en paralelo y reducir el tiempo total de escaneo.

#### Instrucciones

1. Agrega al final de `banner_grabber.py` la función de escaneo paralelo:

```python
# ─────────────────────────────────────────────
#  MOTOR DE ESCANEO PARALELO
# ─────────────────────────────────────────────

from signatures import clasificar_banner  # Importar aquí para evitar circular


def escanear_puerto(host: str, puerto: int, nombre_servicio: str) -> dict:
    """
    Escanea un puerto individual: captura el banner, lo clasifica
    y devuelve un registro estructurado con todos los datos.

    Esta función es ejecutada por cada hilo del ThreadPoolExecutor.

    Args:
        host:            IP del objetivo.
        puerto:          Número de puerto TCP.
        nombre_servicio: Nombre esperado del servicio (para contexto).

    Returns:
        Diccionario con todos los datos del resultado del escaneo.
    """
    timestamp = datetime.utcnow().isoformat() + "Z"

    # Para puertos HTTP usamos requests para obtener cabeceras completas;
    # para el resto usamos sockets TCP directos.
    puertos_http = {80, 443, 8080}

    if puerto in puertos_http:
        http_data = capturar_cabeceras_http(host, puerto)
        banner_raw = http_data["banner"]
        cabeceras  = http_data["headers"]
        http_status = http_data["status"]
    else:
        banner_raw  = capturar_banner_tcp(host, puerto)
        cabeceras   = {}
        http_status = None

    # Clasificar el banner contra el diccionario de firmas
    clasificacion = clasificar_banner(banner_raw)

    return {
        "timestamp":        timestamp,
        "host":             host,
        "port":             puerto,
        "expected_service": nombre_servicio,
        "banner_raw":       banner_raw[:200],  # Truncar para el reporte
        "service":          clasificacion["service"],
        "version":          clasificacion["version"],
        "risk":             clasificacion["risk"],
        "risk_note":        clasificacion["note"],
        "http_status":      http_status,
        "http_headers":     cabeceras,
    }


def ejecutar_escaneo(host: str, puertos: dict,
                     max_workers: int = MAX_WORKERS) -> list:
    """
    Ejecuta el escaneo de todos los puertos en paralelo usando
    ThreadPoolExecutor y devuelve la lista de resultados ordenada por puerto.

    Args:
        host:        IP del objetivo.
        puertos:     Diccionario {puerto: nombre_servicio}.
        max_workers: Número máximo de hilos concurrentes.

    Returns:
        Lista de diccionarios de resultados, ordenada por número de puerto.
    """
    print(f"\n[*] Iniciando escaneo de {len(puertos)} puertos en {host}")
    print(f"[*] Hilos paralelos: {max_workers} | Timeout por puerto: {SOCKET_TIMEOUT}s")
    print("-" * 60)

    resultados = []

    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
        # Crear un futuro por cada puerto
        futuros = {
            executor.submit(escanear_puerto, host, puerto, nombre): (puerto, nombre)
            for puerto, nombre in puertos.items()
        }

        # Recoger resultados conforme se completan
        for futuro in concurrent.futures.as_completed(futuros):
            puerto, nombre = futuros[futuro]
            try:
                resultado = futuro.result()
                resultados.append(resultado)

                # Indicador de progreso en consola
                icono_riesgo = {
                    "CRITICAL": "🔴", "HIGH": "🟠", "MEDIUM": "🟡",
                    "LOW": "🟢", "INFO": "🔵", "UNKNOWN": "⚪",
                }.get(resultado["risk"], "⚪")

                print(
                    f"  {icono_riesgo} Puerto {puerto:>5}/tcp  "
                    f"[{resultado['risk']:<8}]  "
                    f"{resultado['service']} {resultado['version'][:30]}"
                )

            except Exception as e:
                print(f"  ⚠️  Puerto {puerto}: Error al procesar resultado — {e}")
                resultados.append({
                    "port": puerto,
                    "expected_service": nombre,
                    "error": str(e),
                    "risk": "UNKNOWN",
                })

    # Ordenar por número de puerto para presentación consistente
    resultados.sort(key=lambda x: x.get("port", 0))
    return resultados
```

2. Guarda el archivo.

**Verificación de la función de escaneo** (prueba con un solo puerto para no sobrecargar):

```bash
python3 -c "
from banner_grabber import escanear_puerto, TARGET_HOST
r = escanear_puerto(TARGET_HOST, 22, 'SSH')
print('Resultado del escaneo del puerto 22:')
import json
print(json.dumps(r, indent=2, ensure_ascii=False))
"
```

---

### Paso 5: Crear el módulo de generación de reportes

**Objetivo:** Implementar `report.py` para generar el reporte final en JSON y un resumen legible en consola con estadísticas de riesgo.

#### Instrucciones

1. Abre `report.py` y escribe el siguiente contenido:

```python
"""
report.py
Módulo de generación de reportes para la herramienta de banner grabbing.
Lab 04-00-01
"""

import json
import os
from datetime import datetime
from collections import Counter


def generar_reporte_json(resultados: list, host: str,
                          directorio_salida: str = "output") -> str:
    """
    Serializa los resultados del escaneo a un archivo JSON estructurado.

    El archivo se nombra con timestamp para evitar sobreescrituras accidentales.

    Args:
        resultados:        Lista de diccionarios con resultados por puerto.
        host:              IP del objetivo escaneado.
        directorio_salida: Directorio donde guardar el archivo.

    Returns:
        Ruta completa del archivo generado.
    """
    os.makedirs(directorio_salida, exist_ok=True)

    ts = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
    nombre_archivo = f"banner_report_{host.replace('.', '_')}_{ts}.json"
    ruta_completa  = os.path.join(directorio_salida, nombre_archivo)

    # Estructura del reporte completo
    reporte = {
        "metadata": {
            "tool":       "BannerGrabber v1.0 — Lab 04-00-01",
            "target":     host,
            "scan_time":  datetime.utcnow().isoformat() + "Z",
            "total_ports_scanned": len(resultados),
            "authorization": "Entorno de laboratorio controlado — Metasploitable 2",
        },
        "summary": _generar_resumen(resultados),
        "results": resultados,
    }

    with open(ruta_completa, "w", encoding="utf-8") as f:
        json.dump(reporte, f, indent=2, ensure_ascii=False)

    return ruta_completa


def _generar_resumen(resultados: list) -> dict:
    """Genera estadísticas de resumen a partir de la lista de resultados."""
    conteo_riesgo = Counter(r.get("risk", "UNKNOWN") for r in resultados)
    servicios_detectados = [
        f"{r.get('service', 'N/A')} {r.get('version', '')}".strip()
        for r in resultados
        if r.get("service") not in ("N/A", "Unknown", None)
    ]

    return {
        "risk_distribution": dict(conteo_riesgo),
        "critical_count":    conteo_riesgo.get("CRITICAL", 0),
        "high_count":        conteo_riesgo.get("HIGH", 0),
        "services_detected": servicios_detectados,
    }


def imprimir_resumen_consola(resultados: list, host: str) -> None:
    """
    Imprime un resumen formateado en consola con tabla de resultados
    y estadísticas de distribución de riesgo.

    Args:
        resultados: Lista de diccionarios con resultados por puerto.
        host:       IP del objetivo escaneado.
    """
    print("\n" + "═" * 70)
    print(f"  REPORTE DE BANNER GRABBING — Objetivo: {host}")
    print("═" * 70)

    # Cabecera de tabla
    print(f"\n{'Puerto':<8} {'Servicio':<14} {'Versión':<25} {'Riesgo':<10} {'Estado HTTP'}")
    print("-" * 70)

    for r in resultados:
        puerto  = str(r.get("port", "?"))
        servicio = r.get("service", "N/A")[:13]
        version  = r.get("version", "N/A")[:24]
        riesgo   = r.get("risk", "UNKNOWN")[:9]
        http_st  = str(r.get("http_status", "-"))

        print(f"{puerto:<8} {servicio:<14} {version:<25} {riesgo:<10} {http_st}")

    # Estadísticas de riesgo
    resumen = _generar_resumen(resultados)
    dist    = resumen["risk_distribution"]

    print("\n" + "─" * 70)
    print("  DISTRIBUCIÓN DE RIESGO:")
    for nivel, count in sorted(dist.items()):
        barra = "█" * count
        print(f"    {nivel:<10}: {barra} ({count})")

    criticos = resumen["critical_count"]
    altos    = resumen["high_count"]

    if criticos > 0:
        print(f"\n  ⚠️  ATENCIÓN: {criticos} servicio(s) con riesgo CRÍTICO detectado(s).")
    if altos > 0:
        print(f"  ⚠️  {altos} servicio(s) con riesgo ALTO detectado(s).")

    print("\n  SERVICIOS IDENTIFICADOS:")
    for svc in resumen["services_detected"]:
        print(f"    • {svc}")

    print("═" * 70 + "\n")
```

2. Guarda el archivo.

**Verificación del módulo de reporte:**

```bash
python3 -c "
from report import imprimir_resumen_consola
# Datos de prueba simulados
datos_prueba = [
    {'port': 21, 'service': 'ProFTPD', 'version': '1.3.3c', 'risk': 'CRITICAL',
     'http_status': None, 'risk_note': 'Backdoor conocido'},
    {'port': 22, 'service': 'OpenSSH', 'version': '4.7p1', 'risk': 'HIGH',
     'http_status': None, 'risk_note': 'Versión antigua'},
    {'port': 80, 'service': 'Apache', 'version': '2.2.8', 'risk': 'HIGH',
     'http_status': 200, 'risk_note': 'Versión desactualizada'},
]
imprimir_resumen_consola(datos_prueba, '192.168.56.101')
"
```

---

### Paso 6: Integrar todo en el script principal y ejecutar el escaneo completo

**Objetivo:** Completar `banner_grabber.py` con el bloque `main()` que orquesta el escaneo completo, imprime el resumen en consola y guarda el reporte JSON.

#### Instrucciones

1. Agrega al final de `banner_grabber.py` el bloque principal:

```python
# ─────────────────────────────────────────────
#  PUNTO DE ENTRADA PRINCIPAL
# ─────────────────────────────────────────────

# Suprimir advertencias de SSL de requests (cert autofirmado en Metasploitable 2)
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)


def main():
    """
    Función principal que orquesta el escaneo completo:
    1. Valida que el objetivo es alcanzable.
    2. Ejecuta el escaneo paralelo de todos los puertos configurados.
    3. Imprime el resumen en consola.
    4. Guarda el reporte completo en JSON.
    """
    from report import generar_reporte_json, imprimir_resumen_consola

    print("╔══════════════════════════════════════════════════════════════╗")
    print("║      BANNER GRABBER — Lab 04-00-01 (Uso Educativo)          ║")
    print("║      Ejecutar SOLO sobre sistemas con autorización escrita   ║")
    print("╚══════════════════════════════════════════════════════════════╝")
    print(f"\n[*] Objetivo: {TARGET_HOST}")
    print(f"[*] Puertos a escanear: {list(TARGET_PORTS.keys())}")

    # Verificación básica de conectividad antes de escanear
    print("\n[*] Verificando conectividad con el objetivo...")
    try:
        with socket.create_connection((TARGET_HOST, 22), timeout=3):
            print(f"[+] Objetivo {TARGET_HOST} alcanzable (puerto 22 responde).")
    except Exception:
        print(f"[!] No se puede alcanzar {TARGET_HOST}. Verifica que Metasploitable 2")
        print("    esté encendida y en la red NAT interna correcta.")
        return

    # Ejecutar escaneo paralelo
    resultados = ejecutar_escaneo(TARGET_HOST, TARGET_PORTS, MAX_WORKERS)

    # Mostrar resumen en consola
    imprimir_resumen_consola(resultados, TARGET_HOST)

    # Guardar reporte JSON
    ruta_reporte = generar_reporte_json(resultados, TARGET_HOST)
    print(f"[+] Reporte guardado en: {ruta_reporte}")
    print("[*] Escaneo completado.\n")


if __name__ == "__main__":
    main()
```

2. Guarda el archivo. Verifica la estructura final del proyecto:

```bash
ls -la ~/labs/lab04-banner-grabbing/
# Debe mostrar: banner_grabber.py, signatures.py, report.py, output/, venv/
```

3. **Ejecuta el escaneo completo** (asegúrate de que Metasploitable 2 está encendida y en la red NAT interna):

```bash
cd ~/labs/lab04-banner-grabbing
source venv/bin/activate
python3 banner_grabber.py
```

**Salida esperada en consola:**

```
╔══════════════════════════════════════════════════════════════╗
║      BANNER GRABBER — Lab 04-00-01 (Uso Educativo)          ║
║      Ejecutar SOLO sobre sistemas con autorización escrita   ║
╚══════════════════════════════════════════════════════════════╝

[*] Objetivo: 192.168.56.101
[*] Puertos a escanear: [21, 22, 25, 80, 443, 3306, 8080]

[*] Verificando conectividad con el objetivo...
[+] Objetivo 192.168.56.101 alcanzable (puerto 22 responde).

[*] Iniciando escaneo de 7 puertos en 192.168.56.101
[*] Hilos paralelos: 4 | Timeout por puerto: 4.0s
------------------------------------------------------------
  🔴 Puerto    21/tcp  [CRITICAL ]  ProFTPD 1.3.3c
  🔴 Puerto    22/tcp  [HIGH     ]  OpenSSH 4.7p1
  🟠 Puerto    25/tcp  [MEDIUM   ]  Postfix
  🟠 Puerto    80/tcp  [HIGH     ]  Apache 2.2.8
  ⚪ Puerto   443/tcp  [UNKNOWN  ]  N/A N/A
  🔴 Puerto  3306/tcp  [CRITICAL ]  MySQL 5.0.51a
  🟠 Puerto  8080/tcp  [INFO     ]  Apache 2.2.8

══════════════════════════════════════════════════════════════════════
  REPORTE DE BANNER GRABBING — Objetivo: 192.168.56.101
══════════════════════════════════════════════════════════════════════

Puerto   Servicio       Versión                   Riesgo     Estado HTTP
----------------------------------------------------------------------
21       ProFTPD        1.3.3c                    CRITICAL   -
22       OpenSSH        4.7p1                     HIGH       -
25       Postfix        220 metasploitable...      MEDIUM     -
80       Apache         2.2.8                     HIGH       200
443      N/A            N/A                       UNKNOWN    -
3306     MySQL          5.0.51a                   CRITICAL   -
8080     Apache         2.2.8                     HIGH       200

──────────────────────────────────────────────────────────────────────
  DISTRIBUCIÓN DE RIESGO:
    CRITICAL  : ██ (2)
    HIGH      : ███ (3)
    MEDIUM    : █ (1)
    UNKNOWN   : █ (1)

  ⚠️  ATENCIÓN: 2 servicio(s) con riesgo CRÍTICO detectado(s).
  ⚠️  3 servicio(s) con riesgo ALTO detectado(s).

  SERVICIOS IDENTIFICADOS:
    • ProFTPD 1.3.3c
    • OpenSSH 4.7p1
    • Apache 2.2.8
    • MySQL 5.0.51a
══════════════════════════════════════════════════════════════════════

[+] Reporte guardado en: output/banner_report_192_168_56_101_20241015_143022.json
[*] Escaneo completado.
```

---

## 7. Validación y Pruebas

Ejecuta las siguientes verificaciones para confirmar que la herramienta funciona correctamente:

### 7.1 Verificar que el reporte JSON fue generado y es válido

```bash
# Verificar que el archivo existe
ls -lh output/banner_report_*.json

# Validar que el JSON es sintácticamente correcto
python3 -c "
import json, glob, os
archivos = glob.glob('output/banner_report_*.json')
ultimo = max(archivos, key=os.path.getmtime)
with open(ultimo) as f:
    reporte = json.load(f)
print(f'JSON válido: {ultimo}')
print(f'Puertos escaneados: {reporte[\"metadata\"][\"total_ports_scanned\"]}')
print(f'Objetivo: {reporte[\"metadata\"][\"target\"]}')
print(f'Servicios CRITICAL: {reporte[\"summary\"][\"critical_count\"]}')
print(f'Servicios HIGH: {reporte[\"summary\"][\"high_count\"]}')
"
```

**Salida esperada:**

```
JSON válido: output/banner_report_192_168_56_101_20241015_143022.json
Puertos escaneados: 7
Objetivo: 192.168.56.101
Servicios CRITICAL: 2
Servicios HIGH: 3
```

### 7.2 Verificar la clasificación de banners con casos de prueba

```bash
python3 -c "
from signatures import clasificar_banner

casos = [
    ('220 ProFTPD 1.3.3c Server',          'ProFTPD',  'CRITICAL'),
    ('SSH-2.0-OpenSSH_4.7p1 Debian',       'OpenSSH',  'HIGH'),
    ('Apache/2.2.8 (Ubuntu)',               'Apache',   'HIGH'),
    ('5.0.51a-3ubuntu5\x00',               'MySQL',    'CRITICAL'),
    ('',                                    'N/A',      'UNKNOWN'),
]

todos_ok = True
for banner, servicio_esp, riesgo_esp in casos:
    r = clasificar_banner(banner)
    ok = r['service'] == servicio_esp and r['risk'] == riesgo_esp
    estado = '✓ PASS' if ok else '✗ FAIL'
    print(f'{estado} | Banner: {repr(banner[:30]):35s} | '
          f'Esperado: {servicio_esp}/{riesgo_esp} | '
          f'Obtenido: {r[\"service\"]}/{r[\"risk\"]}')
    if not ok:
        todos_ok = False

print()
print('Resultado final:', '✓ TODOS LOS CASOS PASARON' if todos_ok else '✗ HAY FALLOS')
"
```

### 7.3 Verificar que el reporte contiene banners crudos para los puertos abiertos

```bash
python3 -c "
import json, glob, os
archivos = glob.glob('output/banner_report_*.json')
ultimo = max(archivos, key=os.path.getmtime)
with open(ultimo) as f:
    reporte = json.load(f)

print('Banners crudos capturados:')
print('-' * 50)
for r in reporte['results']:
    banner = r.get('banner_raw', '')
    if banner and not banner.startswith('['):
        print(f\"Puerto {r['port']:>5}: {banner[:70]}\")
"
```

---

## 8. Resolución de Problemas

### Problema 1: La herramienta no detecta ningún banner y todos los puertos muestran `[Timeout]`

**Síntomas:**
- Todos los puertos devuelven `[Timeout — sin respuesta]`.
- El mensaje de verificación de conectividad falla con `No se puede alcanzar`.
- `ping` a la IP de Metasploitable 2 no responde.

**Causa probable:**
Las VMs están en adaptadores de red distintos o la red NAT interna no está configurada correctamente. VirtualBox/VMware puede tener la VM de Kali en "NAT" (con salida a Internet) y Metasploitable 2 en "Red interna" o viceversa, sin un adaptador compartido.

**Solución:**

```bash
# Paso 1: Verificar la IP actual de Kali Linux
ip addr show | grep "inet " | grep -v "127.0.0.1"

# Paso 2: Verificar la IP de Metasploitable 2
# (debes verla en la consola de la VM o con el login msfadmin/msfadmin)
# En Metasploitable 2: ifconfig eth0

# Paso 3: Asegurarse de que ambas VMs usan el MISMO adaptador
# En VirtualBox: Configuración de cada VM → Red → Adaptador 1
# → Conectado a: "Red interna" → Nombre: "intnet" (mismo nombre en ambas VMs)

# Paso 4: Si usas "Host-Only Adapter", verificar que el rango coincide
# VirtualBox → Herramientas → Red → Host-Only Networks
# Kali y Metasploitable deben estar en el mismo rango (ej. 192.168.56.0/24)

# Paso 5: Actualizar TARGET_HOST en banner_grabber.py con la IP correcta
grep "TARGET_HOST" banner_grabber.py
# Editar y cambiar la IP si es necesario
```

---

### Problema 2: El puerto 80 muestra `[HTTP Error: ...]` o `service: N/A` aunque Apache está corriendo

**Síntomas:**
- El escaneo TCP del puerto 80 captura datos, pero `capturar_cabeceras_http` falla.
- El resultado del puerto 80 muestra `service: N/A` o `version: N/A` en el reporte JSON.
- En consola aparece un error tipo `ConnectionError` o `RemoteDisconnected`.

**Causa probable:**
La función `capturar_cabeceras_http` usa el método `HEAD`, pero algunos servidores web de Metasploitable 2 (especialmente configuraciones antiguas de Apache 2.2.x) pueden rechazar peticiones `HEAD` o responder de forma no estándar. También puede ocurrir si `requests` no está instalado en el entorno virtual activo.

**Solución:**

```bash
# Paso 1: Verificar que requests está instalado en el entorno virtual activo
which python3
pip show requests | grep Version

# Si no está instalado:
pip install requests==2.31.0

# Paso 2: Probar la conexión HTTP manualmente con curl
curl -I http://192.168.56.101/
# Debe mostrar cabeceras HTTP incluyendo "Server: Apache/2.2.8"

# Paso 3: Si HEAD falla, modificar capturar_cabeceras_http para usar GET
# En banner_grabber.py, cambiar la línea:
#   respuesta = requests.head(url, ...)
# Por:
#   respuesta = requests.get(url, ..., stream=True)
#   respuesta.close()

# Paso 4: Verificar que el banner TCP del puerto 80 sí se captura
python3 -c "
from banner_grabber import capturar_banner_tcp, TARGET_HOST
print(capturar_banner_tcp(TARGET_HOST, 80))
"
# Si devuelve la respuesta HTTP cruda, el fallback TCP funcionará para clasificación
```

---

## 9. Limpieza del Entorno

Una vez completada la práctica, ejecuta los siguientes pasos para dejar el entorno en estado limpio:

```bash
# 1. Desactivar el entorno virtual de Python
deactivate

# 2. Verificar que los reportes JSON fueron guardados correctamente
ls -lh ~/labs/lab04-banner-grabbing/output/

# 3. (Opcional) Comprimir los resultados para entrega
cd ~/labs
tar -czf lab04-banner-grabbing-resultados.tar.gz \
    lab04-banner-grabbing/output/ \
    lab04-banner-grabbing/banner_grabber.py \
    lab04-banner-grabbing/signatures.py \
    lab04-banner-grabbing/report.py

# 4. Apagar Metasploitable 2 o restaurar snapshot
# En VirtualBox:
VBoxManage controlvm "Metasploitable2" poweroff
# O restaurar al snapshot limpio:
# VBoxManage snapshot "Metasploitable2" restore "pre-lab04"

# 5. Verificar que no quedaron procesos de escaneo activos
ps aux | grep banner_grabber
# No debe haber procesos activos

# 6. Revisar que no hay archivos con credenciales o datos sensibles
# en el directorio del proyecto (API keys, contraseñas, etc.)
grep -r "password\|api_key\|secret" ~/labs/lab04-banner-grabbing/ \
    --include="*.py" --include="*.json" 2>/dev/null
```

> **Nota de seguridad:** Los archivos JSON del directorio `output/` contienen información de fingerprinting de la VM objetivo. Aunque se trata de un entorno de laboratorio controlado, trata estos reportes como información sensible y no los compartas fuera del contexto académico.

---

## 10. Resumen

En esta práctica construiste una herramienta de **banner grabbing y fingerprinting activo** completamente funcional en Python, aplicando los conceptos de la Lección 4.1 sobre fingerprinting de servicios. Los puntos clave desarrollados fueron:

| Componente | Técnica aplicada | Módulo Python |
|---|---|---|
| Captura de banners TCP | Sockets raw con sondas por protocolo | `socket` |
| Extracción de cabeceras HTTP | Petición HEAD con manejo de TLS | `requests` |
| Clasificación de servicios | Expresiones regulares + diccionario de firmas | `re` |
| Escaneo paralelo | ThreadPoolExecutor con `as_completed` | `concurrent.futures` |
| Reporte estructurado | Serialización con metadatos y estadísticas | `json` |

La herramienta demostró que Metasploitable 2 expone múltiples servicios con versiones antiguas y vulnerabilidades críticas conocidas (ProFTPD 1.3.3c, vsftpd 2.3.4, MySQL sin autenticación), lo cual es exactamente el tipo de información que un pentester necesita antes de la fase de análisis de vulnerabilidades.

### Conceptos reforzados de la Lección 4.1

- **Fingerprinting activo**: la herramienta envía paquetes al objetivo y analiza respuestas — detectable pero preciso.
- **Banners de bienvenida**: los servicios revelan versiones de forma voluntaria en su mensaje inicial.
- **Comportamiento diferencial**: el uso de sondas específicas por protocolo (HEAD para HTTP, lectura directa para FTP/SSH) ilustra cómo adaptar el estímulo al protocolo objetivo.
- **Fase de enumeración**: los resultados obtenidos son el insumo directo para el análisis de vulnerabilidades en los labs siguientes.

### Recursos adicionales

- [Python `socket` — Documentación oficial](https://docs.python.org/3/library/socket.html)
- [Python `concurrent.futures` — Documentación oficial](https://docs.python.org/3/library/concurrent.futures.html)
- [Requests: HTTP for Humans — Documentación oficial](https://requests.readthedocs.io/)
- [OWASP Testing Guide — OTG-INFO-002: Fingerprint Web Server](https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/01-Information_Gathering/02-Fingerprint_Web_Server)
- [Nmap — Detección de versiones de servicios (`-sV`)](https://nmap.org/book/man-version-detection.html)

---
