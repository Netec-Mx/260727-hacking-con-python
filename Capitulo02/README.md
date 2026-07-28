# Práctica 2 — Script de Recolección Pasiva con APIs Públicas

## 1. Metadatos

| Campo            | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 30 minutos                                   |
| **Complejidad**  | Media                                        |
| **Nivel Bloom**  | Aplicar (*Apply*)                            |
| **Módulo**       | 2 — Reconocimiento Pasivo y OSINT con Python |
| **Lab ID**       | 02-00-01                                     |

---

## 2. Descripción General

En este laboratorio construirás un script Python de reconocimiento pasivo que integra tres fuentes OSINT públicas — Shodan, VirusTotal y WHOIS — para generar un perfil consolidado de un dominio o dirección IP objetivo **sin interactuar directamente con ese objetivo**. Aplicarás los principios estudiados en la Lección 2.1: ninguna consulta se dirigirá al servidor del objetivo; toda la información se obtendrá de terceros de confianza pública. El resultado final será un archivo JSON estructurado y normalizado que consolida los hallazgos de las tres fuentes.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Explicar la diferencia entre reconocimiento pasivo y activo, y justificar por qué el script desarrollado es pasivo.
- [ ] Integrar al menos tres APIs públicas de OSINT (Shodan, VirusTotal, python-whois) en un único script Python modular.
- [ ] Manejar autenticación por API key, límites de tasa (*rate limiting*) y respuestas JSON de servicios externos con manejo de errores robusto.
- [ ] Consolidar resultados de múltiples fuentes en un diccionario estructurado y exportarlo a archivo JSON.
- [ ] Proteger las API keys usando variables de entorno y un archivo `.env` excluido de Git.

---

## 4. Prerrequisitos

### Conocimiento previo

- Haber completado el Lab 01-00-01 o tener un entorno Python 3.10+ configurado con `virtualenv`.
- Comprensión básica de peticiones HTTP (métodos GET, códigos de estado, cabeceras).
- Familiaridad con el formato JSON y con el módulo `json` de Python.
- Haber leído la Lección 2.1 sobre principios del reconocimiento pasivo.

### Acceso y cuentas requeridas

| Servicio        | URL de registro                              | Qué necesitas                          |
|-----------------|----------------------------------------------|----------------------------------------|
| **Shodan**      | https://account.shodan.io/register           | API key gratuita (100 consultas/mes)   |
| **VirusTotal**  | https://www.virustotal.com/gui/join-us        | API key gratuita (4 req/min)           |
| **python-whois**| *(biblioteca local, no requiere cuenta)*     | Instalación vía pip                    |

> ⚠️ **ÉTICA Y LEGALIDAD:** Este script realiza reconocimiento pasivo, lo que significa que nunca envía paquetes al objetivo. Sin embargo, el uso de la información obtenida debe limitarse estrictamente al alcance de un compromiso de auditoría autorizado. No uses este script contra dominios o IPs que no sean de tu propiedad o para los que no tengas autorización escrita explícita. Para este laboratorio, usaremos dominios de ejemplo públicamente conocidos como `scanme.nmap.org` (dominio mantenido por Nmap Project específicamente para pruebas) o un dominio propio.

---

## 5. Entorno de Laboratorio

### Hardware mínimo requerido

| Componente         | Mínimo                          |
|--------------------|---------------------------------|
| CPU                | 64 bits, 2 núcleos              |
| RAM                | 4 GB disponibles para el host   |
| Almacenamiento     | 500 MB libres para el proyecto  |
| Red                | Conexión a Internet (para APIs) |

> **Nota:** Este laboratorio **no requiere** Metasploitable 2 ni red aislada, ya que todo el reconocimiento es pasivo y se realiza desde la máquina host o desde Kali Linux con acceso a Internet. No es necesario crear un snapshot para este laboratorio.

### Software requerido

| Herramienta         | Versión mínima | Verificación                          |
|---------------------|----------------|---------------------------------------|
| Python              | 3.10+          | `python3 --version`                   |
| pip                 | 23+            | `pip --version`                       |
| virtualenv          | 20+            | `virtualenv --version`                |
| requests            | 2.31+          | `pip show requests`                   |
| shodan              | 1.28+          | `pip show shodan`                     |
| python-whois        | 0.8+           | `pip show python-whois`               |
| dnspython           | 2.4+           | `pip show dnspython`                  |
| python-dotenv       | 1.0+           | `pip show python-dotenv`              |

### Preparación del entorno

Ejecuta los siguientes comandos en tu terminal (Kali Linux o sistema host) **antes** de comenzar los pasos del laboratorio:

```bash
# 1. Crear el directorio del proyecto
mkdir -p ~/hacking-toolkit/lab02
cd ~/hacking-toolkit/lab02

# 2. Crear y activar el entorno virtual
python3 -m venv venv
source venv/bin/activate

# 3. Instalar dependencias
pip install requests shodan python-whois dnspython python-dotenv

# 4. Verificar instalaciones
pip list | grep -E "requests|shodan|python-whois|dnspython|python-dotenv"
```

**Salida esperada de verificación:**
```
dnspython          2.4.x
python-dotenv      1.0.x
python-whois       0.8.x
requests           2.31.x
shodan             1.28.x
```

---

## 6. Pasos del Laboratorio

---

### Paso 1: Configurar las API Keys de Forma Segura

**Objetivo:** Almacenar las credenciales de Shodan y VirusTotal en variables de entorno usando un archivo `.env`, garantizando que nunca sean expuestas en el código fuente ni en el repositorio Git.

#### Instrucciones

1. Inicia sesión en [https://account.shodan.io](https://account.shodan.io) y copia tu API key desde la sección **"My Account"**.

2. Inicia sesión en [https://www.virustotal.com/gui/my-apikey](https://www.virustotal.com/gui/my-apikey) y copia tu API key.

3. Crea el archivo `.env` en el directorio del proyecto:

```bash
cd ~/hacking-toolkit/lab02
nano .env
```

4. Agrega el siguiente contenido al archivo `.env` (reemplaza los valores con tus claves reales):

```ini
# Archivo .env — NUNCA commitear este archivo a Git
SHODAN_API_KEY=tu_clave_shodan_aqui
VIRUSTOTAL_API_KEY=tu_clave_virustotal_aqui
```

5. Guarda el archivo (`Ctrl+O`, `Enter`, `Ctrl+X` en nano).

6. Crea el archivo `.gitignore` para proteger las credenciales:

```bash
cat > .gitignore << 'EOF'
# Credenciales y secretos — NUNCA a Git
.env
*.key
secrets.json

# Entorno virtual
venv/
__pycache__/
*.pyc

# Resultados de laboratorio (opcional, pueden contener datos sensibles)
resultados/
EOF
```

7. Verifica que el archivo `.env` existe y tiene el formato correcto:

```bash
cat .env
```

**Salida esperada:**
```
# Archivo .env — NUNCA commitear este archivo a Git
SHODAN_API_KEY=tu_clave_shodan_aqui
VIRUSTOTAL_API_KEY=tu_clave_virustotal_aqui
```

#### Verificación

```bash
# Verificar que .gitignore protege el archivo .env
git init
git status
# El archivo .env NO debe aparecer en "Untracked files"
git status | grep ".env"
# Este comando no debe producir salida (el archivo está ignorado)
```

> ⚠️ **Principio de seguridad:** Las API keys son credenciales de autenticación. Exponerlas en código fuente público es equivalente a publicar una contraseña. El patrón `.env` + `.gitignore` es la práctica estándar de la industria para gestionar secretos en proyectos de desarrollo.

---

### Paso 2: Crear la Estructura Modular del Script

**Objetivo:** Diseñar la arquitectura del script con funciones separadas por fuente de información, siguiendo el principio de responsabilidad única.

#### Instrucciones

1. Crea el archivo principal del script:

```bash
nano ~/hacking-toolkit/lab02/osint_collector.py
```

2. Escribe la estructura base del script con todas las importaciones y la función principal de orquestación:

```python
#!/usr/bin/env python3
"""
osint_collector.py — Script de Reconocimiento Pasivo con APIs Públicas
Lab 02-00-01 | Curso: Hacking Ético con Python

ADVERTENCIA ÉTICA: Este script realiza únicamente reconocimiento PASIVO.
Ninguna consulta se dirige directamente al objetivo. Úsalo exclusivamente
sobre dominios/IPs de tu propiedad o con autorización escrita explícita.
"""

import argparse
import json
import os
import sys
import time
from datetime import datetime
from pathlib import Path

import dns.resolver
import requests
import whois
from dotenv import load_dotenv

# Cargar variables de entorno desde .env
load_dotenv()

# ─── Constantes de configuración ────────────────────────────────────────────
SHODAN_API_KEY = os.getenv("SHODAN_API_KEY", "")
VIRUSTOTAL_API_KEY = os.getenv("VIRUSTOTAL_API_KEY", "")

SHODAN_BASE_URL = "https://api.shodan.io"
VIRUSTOTAL_BASE_URL = "https://www.virustotal.com/api/v3"

# Respetar rate limits: VirusTotal gratuito = 4 req/min
VIRUSTOTAL_DELAY_SECONDS = 15  # Pausa conservadora entre llamadas a VT

# ─── Funciones de utilidad ───────────────────────────────────────────────────

def validar_api_keys() -> bool:
    """
    Verifica que las API keys estén configuradas antes de ejecutar consultas.
    Retorna True si ambas claves están presentes, False en caso contrario.
    """
    errores = []
    if not SHODAN_API_KEY:
        errores.append("SHODAN_API_KEY no está configurada en .env")
    if not VIRUSTOTAL_API_KEY:
        errores.append("VIRUSTOTAL_API_KEY no está configurada en .env")
    
    if errores:
        print("[-] Error de configuración:")
        for error in errores:
            print(f"    • {error}")
        print("\n[!] Crea el archivo .env con tus API keys. Ver Paso 1 del laboratorio.")
        return False
    return True


def resolver_ip(dominio: str) -> str:
    """
    Resuelve el dominio a su dirección IP principal usando DNS público.
    Esta operación es PASIVA: consulta servidores DNS, no el objetivo.
    """
    try:
        respuesta = dns.resolver.resolve(dominio, 'A')
        ip = str(respuesta[0])
        print(f"[+] Resolución DNS: {dominio} → {ip}")
        return ip
    except (dns.resolver.NXDOMAIN, dns.resolver.NoAnswer, Exception) as e:
        print(f"[~] No se pudo resolver {dominio}: {e}")
        return ""


def exportar_json(datos: dict, objetivo: str) -> str:
    """
    Exporta el diccionario de resultados a un archivo JSON con timestamp.
    Retorna la ruta del archivo creado.
    """
    Path("resultados").mkdir(exist_ok=True)
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    # Sanitizar el nombre del objetivo para usarlo en nombre de archivo
    nombre_seguro = objetivo.replace(".", "_").replace("/", "_")
    ruta = f"resultados/osint_{nombre_seguro}_{timestamp}.json"
    
    with open(ruta, "w", encoding="utf-8") as f:
        json.dump(datos, f, indent=2, ensure_ascii=False, default=str)
    
    return ruta


# ─── Funciones de consulta a APIs ────────────────────────────────────────────

def consultar_shodan(ip: str) -> dict:
    """Placeholder — implementado en Paso 3"""
    pass


def consultar_virustotal(objetivo: str) -> dict:
    """Placeholder — implementado en Paso 4"""
    pass


def consultar_whois(dominio: str) -> dict:
    """Placeholder — implementado en Paso 5"""
    pass


# ─── Función principal de orquestación ───────────────────────────────────────

def main():
    parser = argparse.ArgumentParser(
        description="Script de reconocimiento pasivo OSINT — Lab 02-00-01",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Ejemplos de uso:
  python3 osint_collector.py --dominio scanme.nmap.org
  python3 osint_collector.py --ip 45.33.32.156
  python3 osint_collector.py --dominio scanme.nmap.org --solo-whois
        """
    )
    
    grupo = parser.add_mutually_exclusive_group(required=True)
    grupo.add_argument("--dominio", type=str, help="Dominio objetivo (ej: scanme.nmap.org)")
    grupo.add_argument("--ip", type=str, help="Dirección IP objetivo (ej: 45.33.32.156)")
    
    parser.add_argument(
        "--solo-whois",
        action="store_true",
        help="Ejecutar solo la consulta WHOIS (no requiere API keys)"
    )
    parser.add_argument(
        "--salida",
        type=str,
        default=None,
        help="Ruta personalizada para el archivo JSON de salida"
    )
    
    args = parser.parse_args()
    
    print("=" * 60)
    print("  OSINT Collector — Reconocimiento Pasivo")
    print("  Lab 02-00-01 | Hacking Ético con Python")
    print("=" * 60)
    print(f"[*] Inicio: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    
    # Determinar el objetivo principal
    objetivo = args.dominio if args.dominio else args.ip
    es_dominio = args.dominio is not None
    
    print(f"[*] Objetivo: {objetivo}")
    print(f"[*] Tipo: {'Dominio' if es_dominio else 'Dirección IP'}")
    print("-" * 60)
    
    # Estructura de resultados consolidados
    resultados = {
        "metadata": {
            "objetivo": objetivo,
            "tipo": "dominio" if es_dominio else "ip",
            "timestamp": datetime.now().isoformat(),
            "herramienta": "osint_collector.py v1.0",
            "advertencia": "Uso exclusivo en entornos autorizados"
        },
        "dns": {},
        "whois": {},
        "shodan": {},
        "virustotal": {}
    }
    
    # ── Resolución DNS (si se proporcionó dominio) ──
    ip_resuelta = ""
    if es_dominio:
        ip_resuelta = resolver_ip(objetivo)
        resultados["dns"]["ip_principal"] = ip_resuelta
        resultados["dns"]["dominio"] = objetivo
    else:
        ip_resuelta = objetivo
    
    # ── Consulta WHOIS ──
    if es_dominio:
        print("\n[*] Consultando WHOIS...")
        resultados["whois"] = consultar_whois(objetivo)
    
    if args.solo_whois:
        print("\n[!] Modo --solo-whois activado. Omitiendo Shodan y VirusTotal.")
    else:
        # Validar API keys antes de continuar
        if not validar_api_keys():
            print("\n[!] Continuando solo con WHOIS y DNS.")
        else:
            # ── Consulta Shodan ──
            if ip_resuelta:
                print("\n[*] Consultando Shodan...")
                resultados["shodan"] = consultar_shodan(ip_resuelta)
            else:
                print("\n[~] No se puede consultar Shodan sin IP resuelta.")
            
            # ── Pausa para respetar rate limit de VirusTotal ──
            print(f"\n[*] Esperando {VIRUSTOTAL_DELAY_SECONDS}s (rate limit VirusTotal)...")
            time.sleep(VIRUSTOTAL_DELAY_SECONDS)
            
            # ── Consulta VirusTotal ──
            print("[*] Consultando VirusTotal...")
            resultados["virustotal"] = consultar_virustotal(objetivo)
    
    # ── Exportar resultados ──
    print("\n" + "-" * 60)
    print("[*] Exportando resultados...")
    
    if args.salida:
        ruta_salida = args.salida
        with open(ruta_salida, "w", encoding="utf-8") as f:
            json.dump(resultados, f, indent=2, ensure_ascii=False, default=str)
    else:
        ruta_salida = exportar_json(resultados, objetivo)
    
    print(f"[+] Resultados guardados en: {ruta_salida}")
    print(f"[+] Fin: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print("=" * 60)
    
    return resultados


if __name__ == "__main__":
    main()
```

3. Guarda el archivo y verifica que no tiene errores de sintaxis:

```bash
python3 osint_collector.py --help
```

**Salida esperada:**
```
usage: osint_collector.py [-h] (--dominio DOMINIO | --ip IP) [--solo-whois] [--salida SALIDA]

Script de reconocimiento pasivo OSINT — Lab 02-00-01
...
```

#### Verificación

```bash
# Verificar que la estructura de argumentos funciona
python3 osint_collector.py --dominio scanme.nmap.org --solo-whois
# Debe ejecutar sin error (aunque las funciones de API aún son placeholders)
```

---

### Paso 3: Implementar la Consulta a Shodan

**Objetivo:** Agregar la función `consultar_shodan()` que obtiene información de servicios expuestos de una IP usando la API REST de Shodan.

#### Instrucciones

1. Abre el archivo `osint_collector.py` y reemplaza la función `consultar_shodan` (el placeholder del Paso 2) con la siguiente implementación completa:

```python
def consultar_shodan(ip: str) -> dict:
    """
    Consulta la API de Shodan para obtener información sobre una IP.
    
    Shodan es un motor de búsqueda que indexa dispositivos conectados a Internet.
    Esta consulta es PASIVA: Shodan ya escaneó la IP previamente; nosotros
    solo consultamos su base de datos, sin enviar paquetes al objetivo.
    
    Endpoint: GET /shodan/host/{ip}
    Documentación: https://developer.shodan.io/api
    
    Retorna: dict con puertos, servicios, organización, país y OS detectado.
    """
    resultado = {
        "fuente": "Shodan",
        "ip": ip,
        "estado": "sin_datos",
        "puertos_abiertos": [],
        "servicios": [],
        "organizacion": "",
        "pais": "",
        "sistema_operativo": "",
        "hostnames": [],
        "ultima_actualizacion": "",
        "error": None
    }
    
    if not SHODAN_API_KEY:
        resultado["error"] = "API key no configurada"
        resultado["estado"] = "error_configuracion"
        return resultado
    
    url = f"{SHODAN_BASE_URL}/shodan/host/{ip}"
    params = {"key": SHODAN_API_KEY}
    
    try:
        respuesta = requests.get(url, params=params, timeout=15)
        
        # Manejar errores HTTP específicos de Shodan
        if respuesta.status_code == 401:
            resultado["error"] = "API key inválida o sin permisos"
            resultado["estado"] = "error_autenticacion"
            print("    [-] Shodan: API key inválida")
            return resultado
        
        if respuesta.status_code == 404:
            resultado["estado"] = "sin_datos"
            resultado["error"] = "IP no encontrada en la base de datos de Shodan"
            print(f"    [~] Shodan: No hay datos para {ip}")
            return resultado
        
        if respuesta.status_code == 429:
            resultado["error"] = "Rate limit excedido en Shodan"
            resultado["estado"] = "rate_limit"
            print("    [-] Shodan: Rate limit excedido. Espera antes de reintentar.")
            return resultado
        
        respuesta.raise_for_status()
        datos = respuesta.json()
        
        # Extraer y normalizar los campos relevantes
        resultado["estado"] = "ok"
        resultado["puertos_abiertos"] = datos.get("ports", [])
        resultado["organizacion"] = datos.get("org", "Desconocida")
        resultado["pais"] = datos.get("country_name", "Desconocido")
        resultado["sistema_operativo"] = datos.get("os", "No detectado")
        resultado["hostnames"] = datos.get("hostnames", [])
        resultado["ultima_actualizacion"] = datos.get("last_update", "")
        
        # Extraer información de servicios (banners)
        servicios = []
        for item in datos.get("data", []):
            servicio = {
                "puerto": item.get("port"),
                "protocolo": item.get("transport", "tcp"),
                "producto": item.get("product", ""),
                "version": item.get("version", ""),
                "banner_resumen": item.get("data", "")[:200]  # Limitar a 200 chars
            }
            servicios.append(servicio)
        resultado["servicios"] = servicios
        
        # Mostrar resumen en consola
        print(f"    [+] Shodan: {len(resultado['puertos_abiertos'])} puertos abiertos encontrados")
        print(f"    [+] Organización: {resultado['organizacion']}")
        print(f"    [+] País: {resultado['pais']}")
        if resultado["sistema_operativo"] and resultado["sistema_operativo"] != "No detectado":
            print(f"    [+] OS detectado: {resultado['sistema_operativo']}")
        
    except requests.exceptions.Timeout:
        resultado["error"] = "Timeout al conectar con Shodan API"
        resultado["estado"] = "timeout"
        print("    [-] Shodan: Timeout de conexión")
    
    except requests.exceptions.ConnectionError:
        resultado["error"] = "Error de conexión con Shodan API"
        resultado["estado"] = "error_conexion"
        print("    [-] Shodan: Error de conexión. Verifica tu acceso a Internet.")
    
    except requests.exceptions.RequestException as e:
        resultado["error"] = f"Error inesperado: {str(e)}"
        resultado["estado"] = "error_generico"
        print(f"    [-] Shodan: Error inesperado — {e}")
    
    return resultado
```

2. Guarda el archivo.

3. Prueba únicamente la función de Shodan con una IP pública conocida:

```bash
# Prueba rápida con la IP de scanme.nmap.org
python3 - << 'EOF'
import os
from dotenv import load_dotenv
load_dotenv()

# Importar solo la función que acabamos de implementar
import sys
sys.path.insert(0, '.')

# Simular una prueba directa de la función
import requests
import json

SHODAN_API_KEY = os.getenv("SHODAN_API_KEY", "")
ip_prueba = "45.33.32.156"  # scanme.nmap.org

url = f"https://api.shodan.io/shodan/host/{ip_prueba}"
params = {"key": SHODAN_API_KEY}

if SHODAN_API_KEY:
    r = requests.get(url, params=params, timeout=15)
    print(f"Código HTTP: {r.status_code}")
    if r.status_code == 200:
        datos = r.json()
        print(f"Puertos: {datos.get('ports', [])}")
        print(f"País: {datos.get('country_name', 'N/A')}")
    else:
        print(f"Respuesta: {r.text[:200]}")
else:
    print("[-] SHODAN_API_KEY no configurada en .env")
EOF
```

**Salida esperada (con API key válida):**
```
Código HTTP: 200
Puertos: [22, 80, ...]
País: United States
```

#### Verificación

```bash
# Verificar que el script completo no tiene errores de sintaxis con la nueva función
python3 -c "import ast; ast.parse(open('osint_collector.py').read()); print('[+] Sintaxis OK')"
```

---

### Paso 4: Implementar la Consulta a VirusTotal

**Objetivo:** Agregar la función `consultar_virustotal()` que consulta la reputación de un dominio o IP usando la API v3 de VirusTotal.

#### Instrucciones

1. Reemplaza la función `consultar_virustotal` (placeholder) con la siguiente implementación:

```python
def consultar_virustotal(objetivo: str) -> dict:
    """
    Consulta la API v3 de VirusTotal para obtener la reputación de un dominio o IP.
    
    VirusTotal agrega resultados de más de 70 motores antivirus y servicios de
    análisis. Esta consulta es PASIVA: consultamos la base de datos de VT,
    no escaneamos el objetivo directamente.
    
    Rate limit plan gratuito: 4 requests/minuto, 500/día.
    ¡NO ejecutes este script repetidamente para no agotar tu cuota!
    
    Endpoints utilizados:
      - Dominio: GET /api/v3/domains/{domain}
      - IP:      GET /api/v3/ip_addresses/{ip}
    
    Retorna: dict con estadísticas de detección, categorías y reputación.
    """
    resultado = {
        "fuente": "VirusTotal",
        "objetivo": objetivo,
        "tipo": "",
        "estado": "sin_datos",
        "reputacion": 0,
        "estadisticas_deteccion": {},
        "categorias": [],
        "ultima_analisis": "",
        "motores_maliciosos": [],
        "error": None
    }
    
    if not VIRUSTOTAL_API_KEY:
        resultado["error"] = "API key no configurada"
        resultado["estado"] = "error_configuracion"
        return resultado
    
    headers = {
        "x-apikey": VIRUSTOTAL_API_KEY,
        "Accept": "application/json"
    }
    
    # Determinar si el objetivo es IP o dominio
    import re
    patron_ip = r"^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$"
    es_ip = bool(re.match(patron_ip, objetivo))
    
    if es_ip:
        url = f"{VIRUSTOTAL_BASE_URL}/ip_addresses/{objetivo}"
        resultado["tipo"] = "ip"
    else:
        url = f"{VIRUSTOTAL_BASE_URL}/domains/{objetivo}"
        resultado["tipo"] = "dominio"
    
    try:
        respuesta = requests.get(url, headers=headers, timeout=20)
        
        # Manejar errores HTTP específicos de VirusTotal
        if respuesta.status_code == 401:
            resultado["error"] = "API key de VirusTotal inválida"
            resultado["estado"] = "error_autenticacion"
            print("    [-] VirusTotal: API key inválida")
            return resultado
        
        if respuesta.status_code == 404:
            resultado["estado"] = "sin_datos"
            resultado["error"] = f"{objetivo} no encontrado en VirusTotal"
            print(f"    [~] VirusTotal: No hay datos para {objetivo}")
            return resultado
        
        if respuesta.status_code == 429:
            resultado["error"] = "Rate limit excedido en VirusTotal (4 req/min)"
            resultado["estado"] = "rate_limit"
            print("    [-] VirusTotal: Rate limit excedido. Espera 60 segundos.")
            return resultado
        
        respuesta.raise_for_status()
        datos = respuesta.json()
        
        # Navegar la estructura JSON de la respuesta de VT v3
        atributos = datos.get("data", {}).get("attributes", {})
        
        resultado["estado"] = "ok"
        resultado["reputacion"] = atributos.get("reputation", 0)
        resultado["ultima_analisis"] = atributos.get("last_analysis_date", "")
        
        # Estadísticas de análisis de motores
        stats = atributos.get("last_analysis_stats", {})
        resultado["estadisticas_deteccion"] = {
            "malicioso": stats.get("malicious", 0),
            "sospechoso": stats.get("suspicious", 0),
            "inofensivo": stats.get("harmless", 0),
            "sin_deteccion": stats.get("undetected", 0),
            "timeout": stats.get("timeout", 0)
        }
        
        # Categorías asignadas por los motores (solo para dominios)
        categorias_raw = atributos.get("categories", {})
        resultado["categorias"] = list(set(categorias_raw.values()))
        
        # Identificar motores que marcan como malicioso
        resultados_motores = atributos.get("last_analysis_results", {})
        motores_maliciosos = []
        for motor, detalle in resultados_motores.items():
            if detalle.get("category") in ("malicious", "suspicious"):
                motores_maliciosos.append({
                    "motor": motor,
                    "categoria": detalle.get("category"),
                    "resultado": detalle.get("result", "")
                })
        resultado["motores_maliciosos"] = motores_maliciosos
        
        # Mostrar resumen en consola
        maliciosos = resultado["estadisticas_deteccion"]["malicioso"]
        sospechosos = resultado["estadisticas_deteccion"]["sospechoso"]
        total = sum(resultado["estadisticas_deteccion"].values())
        
        print(f"    [+] VirusTotal: {maliciosos}/{total} motores detectan como malicioso")
        print(f"    [+] Reputación: {resultado['reputacion']}")
        
        if maliciosos > 0 or sospechosos > 0:
            print(f"    [!] ALERTA: {maliciosos} malicioso(s), {sospechosos} sospechoso(s)")
        else:
            print(f"    [+] Sin detecciones maliciosas")
    
    except requests.exceptions.Timeout:
        resultado["error"] = "Timeout al conectar con VirusTotal API"
        resultado["estado"] = "timeout"
        print("    [-] VirusTotal: Timeout de conexión")
    
    except requests.exceptions.ConnectionError:
        resultado["error"] = "Error de conexión con VirusTotal API"
        resultado["estado"] = "error_conexion"
        print("    [-] VirusTotal: Error de conexión")
    
    except (requests.exceptions.RequestException, KeyError, ValueError) as e:
        resultado["error"] = f"Error procesando respuesta: {str(e)}"
        resultado["estado"] = "error_generico"
        print(f"    [-] VirusTotal: Error — {e}")
    
    return resultado
```

2. Guarda el archivo.

#### Verificación

```bash
# Verificar sintaxis del archivo completo
python3 -c "import ast; ast.parse(open('osint_collector.py').read()); print('[+] Sintaxis OK')"

# Prueba rápida de la estructura de respuesta de VirusTotal (sin consumir cuota)
python3 - << 'EOF'
import os
from dotenv import load_dotenv
load_dotenv()

VT_KEY = os.getenv("VIRUSTOTAL_API_KEY", "")
if VT_KEY:
    print(f"[+] VirusTotal API key configurada: {VT_KEY[:8]}...")
else:
    print("[-] VIRUSTOTAL_API_KEY no configurada en .env")
EOF
```

> ⚠️ **Rate limiting:** El plan gratuito de VirusTotal permite 4 solicitudes por minuto. El script ya incluye una pausa de 15 segundos (`VIRUSTOTAL_DELAY_SECONDS`) entre la consulta a Shodan y la consulta a VirusTotal. **No ejecutes el script repetidamente** para no agotar tu cuota mensual de 500 consultas/día.

---

### Paso 5: Implementar la Consulta WHOIS

**Objetivo:** Agregar la función `consultar_whois()` que obtiene información de registro de dominio usando `python-whois`, sin requerir API key.

#### Instrucciones

1. Reemplaza la función `consultar_whois` (placeholder) con la siguiente implementación:

```python
def consultar_whois(dominio: str) -> dict:
    """
    Consulta los registros WHOIS de un dominio usando python-whois.
    
    WHOIS es un protocolo de consulta pública que no requiere autenticación.
    Los datos se obtienen de los registradores de dominios (registries).
    Esta operación es PASIVA: no contacta al servidor del objetivo,
    sino a los servidores WHOIS de los registradores.
    
    Nota legal: Los datos de contacto en WHOIS pueden estar protegidos
    por RGPD/GDPR en Europa, resultando en datos enmascarados.
    
    Retorna: dict con registrante, fechas, servidores de nombres y contactos.
    """
    resultado = {
        "fuente": "WHOIS",
        "dominio": dominio,
        "estado": "sin_datos",
        "registrante": "",
        "organizacion": "",
        "pais_registrante": "",
        "registrador": "",
        "fecha_registro": "",
        "fecha_expiracion": "",
        "fecha_actualizacion": "",
        "servidores_de_nombres": [],
        "estado_dominio": [],
        "emails_contacto": [],
        "error": None
    }
    
    try:
        datos_whois = whois.whois(dominio)
        
        if datos_whois is None or not datos_whois.domain_name:
            resultado["estado"] = "sin_datos"
            resultado["error"] = "No se encontraron registros WHOIS para este dominio"
            print(f"    [~] WHOIS: Sin datos para {dominio}")
            return resultado
        
        resultado["estado"] = "ok"
        
        # Normalizar campos (python-whois puede retornar listas o strings)
        def normalizar_campo(valor):
            """Convierte listas a string o retorna el valor directamente."""
            if isinstance(valor, list):
                return valor[0] if valor else ""
            return str(valor) if valor else ""
        
        def normalizar_lista(valor):
            """Asegura que el valor sea una lista de strings."""
            if isinstance(valor, list):
                return [str(v) for v in valor if v]
            elif valor:
                return [str(valor)]
            return []
        
        resultado["registrante"] = normalizar_campo(datos_whois.get("name", ""))
        resultado["organizacion"] = normalizar_campo(datos_whois.get("org", ""))
        resultado["pais_registrante"] = normalizar_campo(datos_whois.get("country", ""))
        resultado["registrador"] = normalizar_campo(datos_whois.get("registrar", ""))
        
        # Fechas
        resultado["fecha_registro"] = normalizar_campo(
            datos_whois.get("creation_date", "")
        )
        resultado["fecha_expiracion"] = normalizar_campo(
            datos_whois.get("expiration_date", "")
        )
        resultado["fecha_actualizacion"] = normalizar_campo(
            datos_whois.get("updated_date", "")
        )
        
        # Listas
        resultado["servidores_de_nombres"] = normalizar_lista(
            datos_whois.get("name_servers", [])
        )
        resultado["estado_dominio"] = normalizar_lista(
            datos_whois.get("status", [])
        )
        resultado["emails_contacto"] = normalizar_lista(
            datos_whois.get("emails", [])
        )
        
        # Mostrar resumen en consola
        print(f"    [+] WHOIS: Dominio registrado por: {resultado['registrador'] or 'Desconocido'}")
        print(f"    [+] Organización: {resultado['organizacion'] or 'No disponible (posible GDPR)'}")
        print(f"    [+] Fecha de registro: {resultado['fecha_registro'] or 'No disponible'}")
        print(f"    [+] Expira: {resultado['fecha_expiracion'] or 'No disponible'}")
        print(f"    [+] Servidores de nombres: {len(resultado['servidores_de_nombres'])} encontrados")
        
    except whois.parser.PywhoisError as e:
        resultado["error"] = f"Error de parseo WHOIS: {str(e)}"
        resultado["estado"] = "error_parseo"
        print(f"    [-] WHOIS: Error de parseo — {e}")
    
    except ConnectionError as e:
        resultado["error"] = f"Error de conexión al servidor WHOIS: {str(e)}"
        resultado["estado"] = "error_conexion"
        print(f"    [-] WHOIS: Error de conexión — {e}")
    
    except Exception as e:
        resultado["error"] = f"Error inesperado: {str(e)}"
        resultado["estado"] = "error_generico"
        print(f"    [-] WHOIS: Error inesperado — {e}")
    
    return resultado
```

2. Guarda el archivo.

3. Prueba la función WHOIS de forma aislada (no requiere API key):

```bash
python3 - << 'EOF'
import whois

dominio_prueba = "scanme.nmap.org"
try:
    datos = whois.whois(dominio_prueba)
    print(f"[+] Registrador: {datos.registrar}")
    print(f"[+] Servidores NS: {datos.name_servers}")
    print(f"[+] Fecha creación: {datos.creation_date}")
    print("[+] WHOIS funciona correctamente")
except Exception as e:
    print(f"[-] Error: {e}")
EOF
```

**Salida esperada:**
```
[+] Registrador: MarkMonitor Inc. (o similar)
[+] Servidores NS: ['ns1.nmap.org', 'ns2.nmap.org'] (o similar)
[+] Fecha creación: 2000-... (o similar)
[+] WHOIS funciona correctamente
```

#### Verificación

```bash
# Ejecutar el script completo en modo solo-whois para validar la función
python3 osint_collector.py --dominio scanme.nmap.org --solo-whois
```

**Salida esperada:**
```
============================================================
  OSINT Collector — Reconocimiento Pasivo
  Lab 02-00-01 | Hacking Ético con Python
============================================================
[*] Inicio: 2024-XX-XX XX:XX:XX
[*] Objetivo: scanme.nmap.org
[*] Tipo: Dominio
------------------------------------------------------------
[+] Resolución DNS: scanme.nmap.org → 45.33.32.156

[*] Consultando WHOIS...
    [+] WHOIS: Dominio registrado por: MarkMonitor Inc.
    [+] Organización: ...
    [+] Fecha de registro: ...
    [+] Expira: ...
    [+] Servidores de nombres: X encontrados

[!] Modo --solo-whois activado. Omitiendo Shodan y VirusTotal.

------------------------------------------------------------
[*] Exportando resultados...
[+] Resultados guardados en: resultados/osint_scanme_nmap_org_XXXXXXXX_XXXXXX.json
[+] Fin: 2024-XX-XX XX:XX:XX
============================================================
```

---

### Paso 6: Ejecutar el Script Completo y Analizar los Resultados

**Objetivo:** Ejecutar el script integrado con las tres fuentes OSINT y verificar la estructura del archivo JSON de salida.

#### Instrucciones

1. Asegúrate de que el entorno virtual está activo y las API keys están en `.env`:

```bash
cd ~/hacking-toolkit/lab02
source venv/bin/activate
cat .env  # Verificar que las claves están presentes
```

2. Ejecuta el script completo contra `scanme.nmap.org`:

```bash
python3 osint_collector.py --dominio scanme.nmap.org
```

> **Nota:** Esta ejecución tardará aproximadamente 20-30 segundos debido a la pausa de rate limiting de VirusTotal. Esto es intencional y correcto.

3. Una vez finalizado, localiza el archivo JSON generado y examina su estructura:

```bash
# Listar archivos generados
ls -la resultados/

# Ver el contenido del JSON con formato legible
python3 -m json.tool resultados/osint_scanme_nmap_org_*.json | head -80
```

4. Extrae estadísticas específicas del JSON usando Python:

```bash
python3 - << 'EOF'
import json
import glob
import os

# Encontrar el archivo más reciente
archivos = sorted(glob.glob("resultados/osint_scanme_nmap_org_*.json"))
if not archivos:
    print("[-] No se encontraron archivos de resultados")
    exit(1)

archivo_reciente = archivos[-1]
print(f"[*] Analizando: {archivo_reciente}\n")

with open(archivo_reciente) as f:
    datos = json.load(f)

# Resumen de metadatos
print("=== METADATOS ===")
meta = datos.get("metadata", {})
print(f"  Objetivo:   {meta.get('objetivo')}")
print(f"  Timestamp:  {meta.get('timestamp')}")

# Resumen DNS
print("\n=== DNS ===")
dns = datos.get("dns", {})
print(f"  IP principal: {dns.get('ip_principal', 'N/A')}")

# Resumen WHOIS
print("\n=== WHOIS ===")
w = datos.get("whois", {})
print(f"  Estado:      {w.get('estado')}")
print(f"  Registrador: {w.get('registrador', 'N/A')}")
print(f"  Expiración:  {w.get('fecha_expiracion', 'N/A')}")
print(f"  NS count:    {len(w.get('servidores_de_nombres', []))}")

# Resumen Shodan
print("\n=== SHODAN ===")
s = datos.get("shodan", {})
print(f"  Estado:      {s.get('estado')}")
print(f"  Puertos:     {s.get('puertos_abiertos', [])}")
print(f"  País:        {s.get('pais', 'N/A')}")
print(f"  Organización:{s.get('organizacion', 'N/A')}")

# Resumen VirusTotal
print("\n=== VIRUSTOTAL ===")
vt = datos.get("virustotal", {})
print(f"  Estado:      {vt.get('estado')}")
stats = vt.get("estadisticas_deteccion", {})
if stats:
    print(f"  Malicioso:   {stats.get('malicioso', 0)}")
    print(f"  Sospechoso:  {stats.get('sospechoso', 0)}")
    print(f"  Inofensivo:  {stats.get('inofensivo', 0)}")
print(f"  Reputación:  {vt.get('reputacion', 'N/A')}")

print("\n[+] Análisis completado.")
EOF
```

**Salida esperada (valores aproximados):**
```
[*] Analizando: resultados/osint_scanme_nmap_org_XXXXXXXX_XXXXXX.json

=== METADATOS ===
  Objetivo:   scanme.nmap.org
  Timestamp:  2024-XX-XXTXX:XX:XX

=== DNS ===
  IP principal: 45.33.32.156

=== WHOIS ===
  Estado:      ok
  Registrador: MarkMonitor Inc.
  Expiración:  202X-XX-XX XX:XX:XX
  NS count:    2

=== SHODAN ===
  Estado:      ok
  Puertos:     [22, 80, 9929, 31337]
  País:        United States
  Organización:Linode

=== VIRUSTOTAL ===
  Estado:      ok
  Malicioso:   0
  Sospechoso:  0
  Inofensivo:  XX
  Reputación:  0

[+] Análisis completado.
```

5. Verifica la estructura completa del JSON exportado:

```bash
# Contar las claves de primer nivel
python3 -c "
import json, glob
f = sorted(glob.glob('resultados/osint_scanme_nmap_org_*.json'))[-1]
d = json.load(open(f))
print('Secciones en el JSON:', list(d.keys()))
print('Total campos en whois:', len(d.get('whois', {})))
print('Total campos en shodan:', len(d.get('shodan', {})))
print('Total campos en virustotal:', len(d.get('virustotal', {})))
"
```

**Salida esperada:**
```
Secciones en el JSON: ['metadata', 'dns', 'whois', 'shodan', 'virustotal']
Total campos en whois: 12
Total campos en shodan: 11
Total campos en virustotal: 10
```

#### Verificación

```bash
# Verificar que el JSON es válido y tiene todas las secciones esperadas
python3 - << 'EOF'
import json, glob, sys

archivos = sorted(glob.glob("resultados/osint_scanme_nmap_org_*.json"))
if not archivos:
    print("[-] ERROR: No se encontró archivo de resultados")
    sys.exit(1)

with open(archivos[-1]) as f:
    datos = json.load(f)

secciones_requeridas = ["metadata", "dns", "whois", "shodan", "virustotal"]
faltantes = [s for s in secciones_requeridas if s not in datos]

if faltantes:
    print(f"[-] FALLO: Secciones faltantes: {faltantes}")
    sys.exit(1)
else:
    print("[+] VALIDACIÓN EXITOSA: JSON contiene todas las secciones requeridas")
    print(f"[+] Archivo: {archivos[-1]}")
    print(f"[+] Tamaño: {len(json.dumps(datos))} bytes")
EOF
```

---

## 7. Validación y Pruebas

Una vez completados todos los pasos, ejecuta las siguientes verificaciones para confirmar que el script funciona correctamente en su totalidad.

### Prueba 1: Verificar la CLI completa

```bash
# Probar con --help
python3 osint_collector.py --help

# Probar que el argumento --ip funciona
python3 osint_collector.py --ip 45.33.32.156 --solo-whois
# Nota: WHOIS no aplica a IPs directamente; debe manejar el caso sin errores
```

### Prueba 2: Verificar manejo de errores con API key inválida

```bash
# Temporalmente sobreescribir la API key con un valor inválido
SHODAN_API_KEY="clave_invalida_para_prueba" \
VIRUSTOTAL_API_KEY="clave_invalida_para_prueba" \
python3 osint_collector.py --dominio scanme.nmap.org
# El script debe continuar y reportar errores de autenticación, no fallar abruptamente
```

**Comportamiento esperado:**
```
[*] Consultando Shodan...
    [-] Shodan: API key inválida
[*] Consultando VirusTotal...
    [-] VirusTotal: API key inválida
[*] Exportando resultados...
[+] Resultados guardados en: resultados/osint_...json
```

### Prueba 3: Verificar que el JSON de error es válido

```bash
# Verificar que incluso con errores el JSON resultante es válido
python3 -c "
import json, glob
archivos = sorted(glob.glob('resultados/osint_scanme_nmap_org_*.json'))
ultimo = archivos[-1]
with open(ultimo) as f:
    datos = json.load(f)
print('[+] JSON válido:', ultimo)
print('[+] Estado Shodan:', datos['shodan'].get('estado'))
print('[+] Estado VT:', datos['virustotal'].get('estado'))
"
```

### Prueba 4: Verificar que .env no es rastreado por Git

```bash
cd ~/hacking-toolkit/lab02
git status 2>/dev/null | grep -E "\.env|\.gitignore"
# .env NO debe aparecer como archivo sin rastrear
# .gitignore SÍ puede aparecer (es seguro commitear el .gitignore)
```

### Lista de verificación final

| Criterio de validación                                              | ¿Cumple? |
|---------------------------------------------------------------------|----------|
| El script acepta `--dominio` e `--ip` como argumentos              | ☐        |
| La función WHOIS retorna datos para `scanme.nmap.org`              | ☐        |
| La función Shodan retorna puertos para `45.33.32.156`              | ☐        |
| La función VirusTotal retorna estadísticas de detección            | ☐        |
| El archivo JSON contiene las 5 secciones requeridas                | ☐        |
| El archivo `.env` está en `.gitignore` y no es rastreado por Git   | ☐        |
| Los errores de API (401, 404, 429) son manejados sin excepciones   | ☐        |
| El script incluye pausa para respetar el rate limit de VirusTotal  | ☐        |

---

## 8. Solución de Problemas

### Problema 1: `ModuleNotFoundError: No module named 'shodan'` (u otro módulo)

**Síntoma:**
```
Traceback (most recent call last):
  File "osint_collector.py", line 15, in <module>
    import shodan
ModuleNotFoundError: No module named 'shodan'
```

**Causa:** El entorno virtual no está activado o las dependencias no fueron instaladas en el entorno correcto. Es un error frecuente cuando se tiene múltiples instalaciones de Python o se olvidó activar el `venv`.

**Solución:**
```bash
# 1. Verificar si el entorno virtual está activo
# El prompt debe mostrar (venv) al inicio
echo $VIRTUAL_ENV
# Si está vacío, el venv no está activo

# 2. Activar el entorno virtual
cd ~/hacking-toolkit/lab02
source venv/bin/activate

# 3. Verificar que Python del venv es el correcto
which python3
# Debe mostrar: .../lab02/venv/bin/python3

# 4. Reinstalar dependencias dentro del venv activo
pip install requests shodan python-whois dnspython python-dotenv

# 5. Verificar instalación
pip list | grep -E "shodan|requests|whois|dns|dotenv"
```

---

### Problema 2: La consulta a Shodan retorna `estado: sin_datos` para una IP conocida

**Síntoma:**
```
[*] Consultando Shodan...
    [~] Shodan: No hay datos para 45.33.32.156
```
El campo `shodan.estado` en el JSON es `"sin_datos"` aunque la IP es pública y conocida.

**Causa:** Existen dos causas posibles: (a) La API key gratuita de Shodan ha agotado sus 100 consultas mensuales, en cuyo caso la API retorna `404` con un mensaje de créditos insuficientes. (b) La IP consultada genuinamente no está en el índice de Shodan (poco probable para IPs públicas populares).

**Solución:**
```bash
# 1. Verificar el estado de los créditos de tu API key
python3 - << 'EOF'
import os, requests
from dotenv import load_dotenv
load_dotenv()

key = os.getenv("SHODAN_API_KEY", "")
if not key:
    print("[-] API key no configurada")
    exit(1)

# Endpoint de información de la API key (no consume créditos de consulta)
r = requests.get(f"https://api.shodan.io/api-info?key={key}", timeout=10)
print(f"Código HTTP: {r.status_code}")
if r.status_code == 200:
    datos = r.json()
    print(f"Plan: {datos.get('plan')}")
    print(f"Créditos de consulta restantes: {datos.get('query_credits')}")
    print(f"Créditos de escaneo restantes: {datos.get('scan_credits')}")
elif r.status_code == 401:
    print("[-] API key inválida")
else:
    print(f"Respuesta: {r.text}")
EOF

# 2. Si los créditos son 0, esperar al próximo mes o usar una IP diferente
# para verificar la funcionalidad del código

# 3. Alternativamente, usar el modo --solo-whois para continuar el lab
python3 osint_collector.py --dominio scanme.nmap.org --solo-whois
```

---

## 9. Limpieza

Al finalizar el laboratorio, realiza los siguientes pasos para mantener el entorno ordenado:

```bash
# 1. Desactivar el entorno virtual
deactivate

# 2. Verificar que no hay API keys en el historial de comandos
# (si usaste las claves directamente en la terminal)
history | grep -E "API_KEY|api_key|shodan|virustotal"
# Si aparecen, limpiar el historial:
# history -c && history -w

# 3. Verificar que .env no fue accidentalmente añadido a Git
cd ~/hacking-toolkit/lab02
git status | grep ".env"
# No debe aparecer ningún resultado

# 4. (Opcional) Comprimir los resultados del laboratorio para entrega
tar -czf lab02_resultados_$(date +%Y%m%d).tar.gz resultados/
ls -lh lab02_resultados_*.tar.gz

# 5. Revisar el espacio usado por el proyecto
du -sh ~/hacking-toolkit/lab02/
```

> **Nota sobre los archivos de resultados:** Los archivos JSON en `resultados/` pueden contener información de dominios consultados. Si estás trabajando en un entorno compartido, considera eliminar estos archivos después de la entrega del laboratorio con `rm -rf resultados/`.

---

## 10. Resumen

En este laboratorio construiste un script de reconocimiento pasivo funcional que integra tres fuentes OSINT en un pipeline Python unificado. Los conceptos clave aplicados fueron:

| Concepto                        | Implementación en el lab                                            |
|---------------------------------|---------------------------------------------------------------------|
| **Reconocimiento pasivo**       | Ninguna función envía paquetes al objetivo; todo pasa por terceros |
| **Integración de APIs**         | Shodan REST API, VirusTotal API v3, python-whois                   |
| **Autenticación por API key**   | Variables de entorno + `python-dotenv` + `.gitignore`              |
| **Manejo de rate limits**       | `time.sleep(15)` antes de VirusTotal; manejo de HTTP 429           |
| **Manejo de errores HTTP**      | Casos específicos para 401, 404, 429 en cada función               |
| **Normalización de resultados** | Diccionario estructurado con campos consistentes por fuente        |
| **Exportación JSON**            | `json.dump` con timestamp en nombre de archivo                     |
| **CLI profesional**             | `argparse` con grupos mutuamente excluyentes y ejemplos de uso     |

### Reflexión ética

El script que construiste es una herramienta de **reconocimiento pasivo legítima** porque:
1. No envía ningún paquete directamente al objetivo.
2. Consulta únicamente servicios de terceros que han recopilado esa información de forma pública.
3. Respeta los límites de uso de las APIs gratuitas.
4. Protege las credenciales de acceso.

Sin embargo, la información consolidada en el JSON puede ser altamente sensible. Recuerda que el **uso de esta información** debe estar siempre dentro del alcance de un compromiso de auditoría autorizado y documentado.

### Recursos adicionales

- [Documentación oficial de Shodan API](https://developer.shodan.io/api)
- [Documentación de VirusTotal API v3](https://developers.virustotal.com/reference/overview)
- [python-whois en PyPI](https://pypi.org/project/python-whois/)
- [PTES — Penetration Testing Execution Standard](http://www.pentest-standard.org/index.php/Intelligence_Gathering)
- [OSINT Framework](https://osintframework.com/) — Mapa interactivo de herramientas OSINT

---
