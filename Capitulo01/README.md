# Práctica 1 — Primer Script para Recolección de Datos

## 1. Metadatos

| Campo            | Valor                        |
|------------------|------------------------------|
| **Duración**     | 24 minutos                   |
| **Complejidad**  | Fácil                        |
| **Nivel Bloom**  | Aplicar (Apply)              |
| **Módulo**       | Capítulo 1 — Lección 1.1     |
| **Modalidad**    | Individual / Kali Linux VM   |

---

## 2. Descripción General

En este laboratorio configurarás el entorno de desarrollo Python que utilizarás durante todo el curso y escribirás tu primer script funcional de recolección de información. Usando exclusivamente módulos estándar de Python (`os`, `sys`, `socket`, `platform`, `subprocess`, `json`), el script obtendrá datos del sistema local —hostname, dirección IP, sistema operativo, puertos abiertos en `localhost` y variables de entorno relevantes— y exportará los resultados a un archivo JSON estructurado.

No se realizan conexiones externas ni se escanean sistemas de terceros. Todo el trabajo ocurre sobre la propia máquina virtual Kali Linux, por lo que este laboratorio no requiere autorización adicional.

---

## 3. Objetivos de Aprendizaje

Al finalizar este laboratorio serás capaz de:

- [ ] Configurar un entorno virtual con `venv` y la estructura de directorios del proyecto que evolucionará durante el curso.
- [ ] Escribir un script Python funcional que recolecte información básica del sistema y la red local usando módulos estándar.
- [ ] Implementar funciones reutilizables con manejo de errores robusto (`try/except`) para operaciones de red y sistema.
- [ ] Organizar datos recolectados en diccionarios anidados y exportarlos correctamente a un archivo JSON.
- [ ] Aplicar las convenciones de sintaxis Python (indentación, docstrings, `if __name__ == "__main__"`) revisadas en la Lección 1.1.

---

## 4. Prerrequisitos

### Conocimientos previos

| Área                          | Nivel requerido                                              |
|-------------------------------|--------------------------------------------------------------|
| Sintaxis Python básica        | Variables, loops, condicionales, definición de funciones     |
| Terminal Linux                | Navegación de directorios, ejecución de comandos básicos     |
| Conceptos de red              | Qué es un hostname, una dirección IP y un puerto TCP         |

### Acceso y herramientas

| Recurso                       | Estado requerido                                             |
|-------------------------------|--------------------------------------------------------------|
| Kali Linux VM (2024.1+)       | Encendida y con sesión iniciada                              |
| Python 3.10+                  | Instalado y accesible como `python3`                         |
| `pip` y `venv`                | Disponibles (incluidos con Python 3.10+)                     |
| IDE (VS Code o terminal)      | Abierto dentro de la VM                                      |
| Git                           | Instalado (`git --version` debe responder)                   |

> **Nota ética y legal:** Este laboratorio opera exclusivamente sobre tu propia máquina virtual. No se realizan conexiones a sistemas externos. No se requiere formulario de autorización adicional para esta práctica.

---

## 5. Entorno de Laboratorio

### Hardware mínimo del host

| Componente        | Mínimo recomendado                                    |
|-------------------|-------------------------------------------------------|
| RAM               | 8 GB host (4 GB asignados a la VM)                    |
| CPU               | 64 bits con virtualización habilitada (VT-x / AMD-V)  |
| Almacenamiento    | 60 GB libres para la VM y snapshots                   |

### Software dentro de la VM Kali Linux

| Herramienta       | Versión mínima   | Verificación                    |
|-------------------|------------------|---------------------------------|
| Python            | 3.10+            | `python3 --version`             |
| pip               | 23+              | `pip3 --version`                |
| venv              | incluido         | `python3 -m venv --help`        |
| Git               | 2.x              | `git --version`                 |

### Preparación inicial del entorno (ejecutar una sola vez)

Abre una terminal en Kali Linux y ejecuta los siguientes comandos para verificar que todo está disponible:

```bash
# 1. Verificar Python
python3 --version

# 2. Verificar pip
pip3 --version

# 3. Verificar Git
git --version

# 4. Verificar que venv está disponible
python3 -m venv --help | head -3
```

Si alguno de los comandos falla, instala la herramienta faltante antes de continuar:

```bash
# Instalar pip y venv si no están disponibles
sudo apt update && sudo apt install -y python3-pip python3-venv git
```

---

## 6. Procedimiento Paso a Paso

---

### Paso 1 — Crear la Estructura de Directorios del Proyecto

**Objetivo:** Establecer la estructura de carpetas que se usará durante todo el curso y que seguirá creciendo en laboratorios posteriores.

#### Instrucciones

1. Abre una terminal en Kali Linux.

2. Navega a tu directorio de trabajo (o créalo si no existe):

```bash
mkdir -p ~/curso_hacking_python
cd ~/curso_hacking_python
```

3. Crea la estructura completa del proyecto con un solo bloque de comandos:

```bash
mkdir -p toolkit/{utils,data,scripts,tests}
touch toolkit/__init__.py
touch toolkit/utils/__init__.py
touch toolkit/data/.gitkeep
touch toolkit/tests/.gitkeep
```

4. Crea los archivos de configuración del proyecto:

```bash
# Archivo de dependencias (vacío por ahora, se llenará en labs posteriores)
touch toolkit/requirements.txt

# Archivo de ignorados para Git
cat > toolkit/.gitignore << 'EOF'
# Entorno virtual
.venv/
venv/

# Datos sensibles y resultados
data/*.json
data/*.csv
*.log

# Claves API (NUNCA versionar)
.env
secrets.py

# Caché de Python
__pycache__/
*.pyc
*.pyo

# IDE
.vscode/
.idea/
EOF
```

5. Verifica la estructura creada:

```bash
find toolkit/ -type f -o -type d | sort
```

#### Salida esperada

```
toolkit/
toolkit/.gitignore
toolkit/__init__.py
toolkit/data
toolkit/data/.gitkeep
toolkit/requirements.txt
toolkit/scripts
toolkit/tests
toolkit/tests/.gitkeep
toolkit/utils
toolkit/utils/__init__.py
```

#### Verificación

```bash
# Confirmar que la estructura existe y está completa
ls -la toolkit/
ls -la toolkit/utils/
```

Debes ver los directorios `data/`, `scripts/`, `tests/`, `utils/` y los archivos `__init__.py`, `requirements.txt` y `.gitignore`.

---

### Paso 2 — Configurar el Entorno Virtual

**Objetivo:** Crear y activar un entorno virtual Python aislado dentro del proyecto, siguiendo el patrón de la Lección 1.1.

#### Instrucciones

1. Entra al directorio del proyecto:

```bash
cd ~/curso_hacking_python/toolkit
```

2. Crea el entorno virtual usando `venv`:

```bash
python3 -m venv .venv
```

3. Activa el entorno virtual:

```bash
source .venv/bin/activate
```

4. Verifica que el entorno está activo. El prompt debe mostrar `(.venv)` al inicio:

```bash
# Verificar que Python apunta al entorno virtual
which python
python --version
```

5. Actualiza `pip` dentro del entorno virtual:

```bash
pip install --upgrade pip
```

6. Confirma que no hay dependencias externas instaladas aún (solo las de stdlib):

```bash
pip list
```

#### Salida esperada

```
(.venv) kali@kali:~/curso_hacking_python/toolkit$
which python
# /home/kali/curso_hacking_python/toolkit/.venv/bin/python

python --version
# Python 3.11.x  (o 3.10.x según tu instalación)

pip list
# Package    Version
# ---------- -------
# pip        24.x.x
```

#### Verificación

```bash
# El path de Python debe contener ".venv"
python -c "import sys; print(sys.executable)"
```

La salida debe mostrar una ruta que contenga `.venv/bin/python`. Si muestra `/usr/bin/python3`, el entorno **no está activado** — repite el paso 3.

---

### Paso 3 — Inicializar el Repositorio Git

**Objetivo:** Versionar el proyecto desde el primer momento, aplicando buenas prácticas que se mantendrán durante todo el curso.

#### Instrucciones

1. Asegúrate de estar en el directorio `toolkit/` con el entorno virtual activo:

```bash
cd ~/curso_hacking_python/toolkit
# El prompt debe mostrar (.venv)
```

2. Inicializa el repositorio Git:

```bash
git init
```

3. Configura tu identidad en Git (solo si no lo has hecho antes):

```bash
git config user.name "Tu Nombre"
git config user.email "tu@email.com"
```

4. Agrega los archivos iniciales y realiza el primer commit:

```bash
git add .gitignore requirements.txt __init__.py utils/__init__.py
git commit -m "feat: estructura inicial del proyecto toolkit"
```

5. Verifica el estado del repositorio:

```bash
git log --oneline
git status
```

#### Salida esperada

```
Initialized empty Git repository in .../toolkit/.git/
[main (root-commit) abc1234] feat: estructura inicial del proyecto toolkit
 4 files changed, 12 insertions(+)
```

#### Verificación

```bash
git log --oneline
# Debe mostrar al menos 1 commit
```

> **Importante:** Observa que `.venv/` y `data/*.json` **no aparecen** en `git status` gracias al `.gitignore`. Esto es correcto y esperado.

---

### Paso 4 — Escribir el Script de Recolección de Datos

**Objetivo:** Desarrollar el script principal `recolector_sistema.py` que implementa todas las funciones de recolección de información del sistema local.

#### Instrucciones

1. Crea el archivo del script en el directorio `scripts/`:

```bash
# Asegúrate de estar en toolkit/ con (.venv) activo
nano scripts/recolector_sistema.py
```

2. Escribe el siguiente código completo en el archivo. Léelo detenidamente — cada sección está comentada para que entiendas su propósito:

```python
#!/usr/bin/env python3
"""
recolector_sistema.py
---------------------
Script de recolección de información básica del sistema y red local.
Usa exclusivamente módulos estándar de Python (sin dependencias externas).

Curso: Hacking con Python — Capítulo 1, Lab 01-00-01
ADVERTENCIA: Este script opera únicamente sobre el sistema local.
             No realiza conexiones externas de ningún tipo.
"""

import os
import sys
import socket
import platform
import subprocess
import json
from datetime import datetime


# ──────────────────────────────────────────────
# FUNCIONES DE RECOLECCIÓN
# ──────────────────────────────────────────────

def obtener_info_sistema() -> dict:
    """
    Recolecta información básica del sistema operativo.

    Retorna:
        dict: Diccionario con hostname, OS, versión, arquitectura y usuario.
    """
    info = {}
    try:
        info["hostname"]       = socket.gethostname()
        info["sistema_op"]     = platform.system()
        info["version_os"]     = platform.version()
        info["release_os"]     = platform.release()
        info["arquitectura"]   = platform.machine()
        info["procesador"]     = platform.processor()
        info["python_version"] = sys.version
        info["usuario_actual"] = os.getenv("USER", os.getenv("USERNAME", "desconocido"))
    except Exception as e:
        info["error"] = f"Error al obtener info del sistema: {str(e)}"
    return info


def obtener_info_red() -> dict:
    """
    Recolecta información de red del sistema local.
    Obtiene la IP local resolviendo el hostname, y el FQDN.

    Retorna:
        dict: Diccionario con ip_local, hostname y fqdn.
    """
    info = {}
    try:
        hostname = socket.gethostname()
        info["hostname"] = hostname
        info["fqdn"]     = socket.getfqdn()

        # Obtener IP local conectando a un destino externo (sin enviar datos)
        # Este truco usa UDP para descubrir la interfaz de salida sin
        # establecer una conexión real.
        with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
            s.connect(("8.8.8.8", 80))   # No envía datos; solo configura la ruta
            info["ip_local"] = s.getsockname()[0]

    except OSError as e:
        info["ip_local"] = "127.0.0.1"
        info["error_red"] = f"No se pudo determinar IP local: {str(e)}"
    except Exception as e:
        info["error_red"] = f"Error inesperado en red: {str(e)}"
    return info


def escanear_puertos_localhost(puertos: list[int], timeout: float = 0.5) -> dict:
    """
    Verifica qué puertos de la lista están abiertos en localhost (127.0.0.1).

    Parámetros:
        puertos  (list[int]): Lista de números de puerto a verificar.
        timeout  (float):     Tiempo máximo de espera por puerto en segundos.

    Retorna:
        dict: Con listas "abiertos" y "cerrados" de puertos.
    """
    resultado = {"abiertos": [], "cerrados": []}

    for puerto in puertos:
        try:
            with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
                s.settimeout(timeout)
                codigo = s.connect_ex(("127.0.0.1", puerto))
                if codigo == 0:
                    resultado["abiertos"].append(puerto)
                else:
                    resultado["cerrados"].append(puerto)
        except socket.error as e:
            # Si hay un error de socket, consideramos el puerto no disponible
            resultado["cerrados"].append(puerto)

    return resultado


def obtener_variables_entorno(claves_interes: list[str]) -> dict:
    """
    Extrae variables de entorno relevantes para el contexto de seguridad.
    Solo extrae las claves especificadas, nunca el entorno completo.

    Parámetros:
        claves_interes (list[str]): Lista de nombres de variables a extraer.

    Retorna:
        dict: Variables encontradas con sus valores; "NO_DEFINIDA" si no existen.
    """
    variables = {}
    for clave in claves_interes:
        variables[clave] = os.getenv(clave, "NO_DEFINIDA")
    return variables


def obtener_interfaces_red() -> list[str]:
    """
    Obtiene la lista de interfaces de red disponibles en el sistema
    usando el comando 'ip link show' (Linux) o 'ipconfig' (Windows).

    Retorna:
        list[str]: Lista de líneas de salida del comando, o mensaje de error.
    """
    try:
        if platform.system() == "Windows":
            comando = ["ipconfig"]
        else:
            comando = ["ip", "link", "show"]

        resultado = subprocess.run(
            comando,
            capture_output=True,
            text=True,
            timeout=5
        )

        if resultado.returncode == 0:
            # Retornar solo líneas no vacías
            lineas = [l.strip() for l in resultado.stdout.splitlines() if l.strip()]
            return lineas
        else:
            return [f"Error al ejecutar comando: {resultado.stderr.strip()}"]

    except FileNotFoundError:
        return ["Comando no encontrado en este sistema."]
    except subprocess.TimeoutExpired:
        return ["El comando excedió el tiempo de espera."]
    except Exception as e:
        return [f"Error inesperado: {str(e)}"]


# ──────────────────────────────────────────────
# FUNCIÓN DE EXPORTACIÓN
# ──────────────────────────────────────────────

def exportar_json(datos: dict, ruta_archivo: str) -> bool:
    """
    Exporta el diccionario de resultados a un archivo JSON con formato legible.

    Parámetros:
        datos         (dict): Datos a exportar.
        ruta_archivo  (str):  Ruta del archivo de destino.

    Retorna:
        bool: True si la exportación fue exitosa, False en caso contrario.
    """
    try:
        # Crear el directorio de destino si no existe
        directorio = os.path.dirname(ruta_archivo)
        if directorio:
            os.makedirs(directorio, exist_ok=True)

        with open(ruta_archivo, "w", encoding="utf-8") as archivo:
            json.dump(datos, archivo, indent=4, ensure_ascii=False)

        print(f"[+] Resultados exportados a: {ruta_archivo}")
        return True

    except PermissionError:
        print(f"[-] Sin permisos para escribir en: {ruta_archivo}")
        return False
    except OSError as e:
        print(f"[-] Error de sistema al escribir archivo: {str(e)}")
        return False


# ──────────────────────────────────────────────
# FUNCIÓN PRINCIPAL
# ──────────────────────────────────────────────

def main() -> None:
    """
    Orquesta la recolección de datos del sistema y exporta los resultados.
    """
    print("=" * 55)
    print("  Recolector de Información del Sistema — Lab 01-00-01")
    print("=" * 55)

    # Timestamp de inicio
    timestamp = datetime.now().isoformat()
    print(f"[*] Iniciando recolección: {timestamp}\n")

    # ── 1. Información del sistema operativo ──
    print("[*] Recolectando información del sistema...")
    info_sistema = obtener_info_sistema()
    print(f"    Hostname    : {info_sistema.get('hostname', 'N/A')}")
    print(f"    Sistema OS  : {info_sistema.get('sistema_op', 'N/A')} "
          f"{info_sistema.get('release_os', '')}")
    print(f"    Arquitectura: {info_sistema.get('arquitectura', 'N/A')}")

    # ── 2. Información de red ──
    print("\n[*] Recolectando información de red...")
    info_red = obtener_info_red()
    print(f"    IP Local    : {info_red.get('ip_local', 'N/A')}")
    print(f"    FQDN        : {info_red.get('fqdn', 'N/A')}")

    # ── 3. Escaneo de puertos en localhost ──
    # Lista de puertos comunes a verificar en el sistema local
    puertos_a_verificar = [22, 80, 443, 3306, 5432, 8080, 8443]
    print(f"\n[*] Verificando puertos en localhost: {puertos_a_verificar}")
    info_puertos = escanear_puertos_localhost(puertos_a_verificar, timeout=0.3)
    print(f"    Puertos abiertos : {info_puertos['abiertos']}")
    print(f"    Puertos cerrados : {info_puertos['cerrados']}")

    # ── 4. Variables de entorno relevantes ──
    claves_env = ["PATH", "HOME", "SHELL", "LANG", "TERM", "VIRTUAL_ENV"]
    print("\n[*] Extrayendo variables de entorno relevantes...")
    info_env = obtener_variables_entorno(claves_env)
    for clave, valor in info_env.items():
        # Truncar valores muy largos para la impresión en pantalla
        valor_display = valor[:60] + "..." if len(valor) > 60 else valor
        print(f"    {clave:<15}: {valor_display}")

    # ── 5. Interfaces de red ──
    print("\n[*] Obteniendo interfaces de red del sistema...")
    interfaces = obtener_interfaces_red()
    # Mostrar solo las primeras 10 líneas para no saturar la terminal
    for linea in interfaces[:10]:
        print(f"    {linea}")
    if len(interfaces) > 10:
        print(f"    ... ({len(interfaces) - 10} líneas adicionales en el JSON)")

    # ── 6. Ensamblar resultado final ──
    resultado_final = {
        "metadata": {
            "herramienta": "recolector_sistema.py",
            "version":     "1.0.0",
            "lab":         "01-00-01",
            "timestamp":   timestamp,
            "advertencia": "Datos del sistema local — solo para uso educativo"
        },
        "sistema":    info_sistema,
        "red":        info_red,
        "puertos":    info_puertos,
        "entorno":    info_env,
        "interfaces": interfaces
    }

    # ── 7. Exportar a JSON ──
    ruta_salida = os.path.join(
        os.path.dirname(os.path.abspath(__file__)),
        "..", "data", "recoleccion_sistema.json"
    )
    print("\n[*] Exportando resultados...")
    exito = exportar_json(resultado_final, ruta_salida)

    # ── 8. Resumen final ──
    print("\n" + "=" * 55)
    if exito:
        print("  [✓] Recolección completada exitosamente.")
    else:
        print("  [!] Recolección completada con errores de exportación.")
    print("=" * 55)


# ──────────────────────────────────────────────
# PUNTO DE ENTRADA
# ──────────────────────────────────────────────

if __name__ == "__main__":
    main()
```

3. Guarda el archivo (`Ctrl+O`, `Enter`, `Ctrl+X` en nano; o `Ctrl+S` en VS Code).

#### Salida esperada

No hay salida en este paso — solo se crea el archivo. Continúa al siguiente paso para ejecutarlo.

#### Verificación

```bash
# Verificar que el archivo existe y tiene contenido
ls -lh scripts/recolector_sistema.py
wc -l scripts/recolector_sistema.py
# Debe tener aproximadamente 180+ líneas
```

---

### Paso 5 — Ejecutar el Script y Verificar los Resultados

**Objetivo:** Ejecutar el script de recolección, interpretar la salida en pantalla y confirmar que el archivo JSON se generó correctamente.

#### Instrucciones

1. Asegúrate de estar en `toolkit/` con el entorno virtual activo (`(.venv)` en el prompt):

```bash
cd ~/curso_hacking_python/toolkit
source .venv/bin/activate   # Si no está ya activo
```

2. Ejecuta el script:

```bash
python scripts/recolector_sistema.py
```

3. Observa la salida completa en pantalla. Debería mostrar información real de tu sistema.

4. Verifica que el archivo JSON fue creado:

```bash
ls -lh data/recoleccion_sistema.json
```

5. Examina el contenido del JSON con formato legible:

```bash
python -m json.tool data/recoleccion_sistema.json
```

6. Extrae valores específicos usando Python en modo interactivo para validar la estructura:

```bash
python3 -c "
import json
with open('data/recoleccion_sistema.json') as f:
    datos = json.load(f)

print('=== Validación de estructura JSON ===')
print('Claves principales:', list(datos.keys()))
print('Hostname:', datos['sistema']['hostname'])
print('IP Local:', datos['red']['ip_local'])
print('Puertos abiertos:', datos['puertos']['abiertos'])
print('Timestamp:', datos['metadata']['timestamp'])
"
```

#### Salida esperada

La salida en pantalla debe ser similar a (los valores variarán según tu sistema):

```
=======================================================
  Recolector de Información del Sistema — Lab 01-00-01
=======================================================
[*] Iniciando recolección: 2024-11-15T10:23:45.123456

[*] Recolectando información del sistema...
    Hostname    : kali
    Sistema OS  : Linux 6.6.9-amd64
    Arquitectura: x86_64

[*] Recolectando información de red...
    IP Local    : 10.0.2.15
    FQDN        : kali

[*] Verificando puertos en localhost: [22, 80, 443, 3306, 5432, 8080, 8443]
    Puertos abiertos : [22]
    Puertos cerrados : [80, 443, 3306, 5432, 8080, 8443]

[*] Extrayendo variables de entorno relevantes...
    PATH           : /home/kali/.venv/bin:/usr/local/sbin:/usr/local/bin...
    HOME           : /home/kali
    SHELL          : /bin/bash
    LANG           : en_US.UTF-8
    TERM           : xterm-256color
    VIRTUAL_ENV    : /home/kali/curso_hacking_python/toolkit/.venv

[*] Obteniendo interfaces de red del sistema...
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue ...
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...

[*] Exportando resultados...
[+] Resultados exportados a: .../data/recoleccion_sistema.json

=======================================================
  [✓] Recolección completada exitosamente.
=======================================================
```

#### Verificación

```bash
# El archivo JSON debe existir y tener más de 500 bytes
ls -lh data/recoleccion_sistema.json

# Debe ser JSON válido (sin errores)
python -m json.tool data/recoleccion_sistema.json > /dev/null && echo "JSON válido ✓"
```

---

### Paso 6 — Versionar el Script con Git

**Objetivo:** Registrar el trabajo completado en el repositorio Git, aplicando convenciones de mensajes de commit semánticos.

#### Instrucciones

1. Revisa qué archivos nuevos existen en el repositorio:

```bash
git status
```

2. Agrega el script al área de staging:

```bash
git add scripts/recolector_sistema.py
```

3. Verifica que el archivo JSON **no** está siendo rastreado (gracias al `.gitignore`):

```bash
git status
# data/recoleccion_sistema.json NO debe aparecer en la lista
```

4. Realiza el commit del script:

```bash
git commit -m "feat(lab01): agregar script de recolección de información del sistema"
```

5. Verifica el historial de commits:

```bash
git log --oneline
```

#### Salida esperada

```
git status
# On branch main
# Changes to be committed:
#   new file:   scripts/recolector_sistema.py
# (data/ no aparece porque está en .gitignore)

git log --oneline
# def5678 feat(lab01): agregar script de recolección de información del sistema
# abc1234 feat: estructura inicial del proyecto toolkit
```

#### Verificación

```bash
# Confirmar que hay exactamente 2 commits
git log --oneline | wc -l
# Debe mostrar: 2
```

---

## 7. Validación y Pruebas

Ejecuta las siguientes verificaciones para confirmar que el laboratorio está completo y correcto:

```bash
# ── TEST 1: Entorno virtual activo ──
echo "TEST 1: Entorno virtual"
python -c "import sys; ruta = sys.executable; print('[PASS]' if '.venv' in ruta else '[FAIL]', 'Python en:', ruta)"

# ── TEST 2: Módulos estándar disponibles ──
echo "TEST 2: Módulos estándar"
python -c "
import os, sys, socket, platform, subprocess, json
from datetime import datetime
print('[PASS] Todos los módulos estándar importados correctamente')
"

# ── TEST 3: Script ejecuta sin errores ──
echo "TEST 3: Ejecución del script"
python scripts/recolector_sistema.py > /dev/null 2>&1 && echo "[PASS] Script ejecutado sin errores" || echo "[FAIL] El script produjo errores"

# ── TEST 4: JSON generado y válido ──
echo "TEST 4: Archivo JSON"
python -c "
import json, os
ruta = 'data/recoleccion_sistema.json'
if os.path.exists(ruta):
    with open(ruta) as f:
        datos = json.load(f)
    claves = set(datos.keys())
    esperadas = {'metadata', 'sistema', 'red', 'puertos', 'entorno', 'interfaces'}
    if esperadas.issubset(claves):
        print('[PASS] JSON válido con todas las claves esperadas')
    else:
        print('[FAIL] Faltan claves:', esperadas - claves)
else:
    print('[FAIL] Archivo JSON no encontrado')
"

# ── TEST 5: Estructura de directorios ──
echo "TEST 5: Estructura del proyecto"
python -c "
import os
dirs_requeridos = ['utils', 'data', 'scripts', 'tests', '.venv']
archivos_requeridos = ['requirements.txt', '.gitignore', '__init__.py']
base = '.'
todos_ok = True
for d in dirs_requeridos:
    ruta = os.path.join(base, d)
    if not os.path.isdir(ruta):
        print(f'[FAIL] Directorio faltante: {d}')
        todos_ok = False
for a in archivos_requeridos:
    ruta = os.path.join(base, a)
    if not os.path.isfile(ruta):
        print(f'[FAIL] Archivo faltante: {a}')
        todos_ok = False
if todos_ok:
    print('[PASS] Estructura del proyecto completa')
"

# ── TEST 6: Git con 2 commits ──
echo "TEST 6: Historial Git"
commits=$(git log --oneline | wc -l)
[ "$commits" -ge 2 ] && echo "[PASS] Git tiene $commits commits" || echo "[FAIL] Se esperaban al menos 2 commits, hay $commits"
```

**Resultado esperado:** Todos los tests deben mostrar `[PASS]`. Si alguno muestra `[FAIL]`, revisa el paso correspondiente.

---

## 8. Resolución de Problemas

### Problema 1 — El script falla con `ModuleNotFoundError` al importar módulos estándar

**Síntoma:**
```
ModuleNotFoundError: No module named 'socket'
```
O cualquier módulo estándar como `os`, `platform`, `json`, etc.

**Causa:**
El entorno virtual está activo pero apunta a una instalación de Python incompleta o corrupta. Esto puede ocurrir si el entorno fue creado con una versión de Python que no incluye la biblioteca estándar completa (raro, pero posible en instalaciones mínimas de Linux).

**Solución:**

```bash
# 1. Desactivar el entorno virtual actual
deactivate

# 2. Verificar que Python del sistema tiene la stdlib completa
python3 -c "import socket, os, platform, json; print('stdlib OK')"

# 3. Si el paso anterior falla, reinstalar Python
sudo apt install --reinstall python3 python3-stdlib-extensions

# 4. Eliminar el entorno virtual corrupto y recrearlo
rm -rf .venv
python3 -m venv .venv
source .venv/bin/activate

# 5. Verificar nuevamente
python -c "import socket, os, platform, json; print('stdlib OK en venv')"
```

---

### Problema 2 — El archivo JSON no se crea y aparece `PermissionError` o la ruta es incorrecta

**Síntoma:**
```
[-] Sin permisos para escribir en: .../data/recoleccion_sistema.json
```
O el script termina con `[✓]` pero el archivo no aparece en `data/`.

**Causa:**
El script calcula la ruta del directorio `data/` de forma relativa al archivo `recolector_sistema.py`. Si el script se ejecuta desde un directorio diferente a `toolkit/`, la ruta resultante puede apuntar a un lugar incorrecto o sin permisos de escritura.

**Solución:**

```bash
# 1. Verificar desde dónde se está ejecutando el script
pwd
# Debe mostrar: .../toolkit/

# 2. Ejecutar SIEMPRE desde el directorio toolkit/
cd ~/curso_hacking_python/toolkit
python scripts/recolector_sistema.py

# 3. Si el problema persiste, verificar permisos del directorio data/
ls -la data/
# Debe tener permisos de escritura para tu usuario

# 4. Si data/ no tiene permisos, corregirlos
chmod 755 data/

# 5. Alternativa: ejecutar el script con ruta absoluta explícita
python $(pwd)/scripts/recolector_sistema.py
```

---

## 9. Limpieza

Al finalizar el laboratorio, realiza las siguientes acciones para dejar el entorno ordenado:

```bash
# 1. Asegurarse de estar en el directorio correcto
cd ~/curso_hacking_python/toolkit

# 2. Verificar que el JSON de resultados está en data/ (no en otro lugar)
ls -lh data/recoleccion_sistema.json

# 3. Desactivar el entorno virtual
deactivate

# 4. Verificar que el prompt ya NO muestra (.venv)
# El prompt debe volver a: kali@kali:~/curso_hacking_python/toolkit$

# 5. Confirmar el estado final del repositorio Git
git status
# Debe mostrar: "nothing to commit, working tree clean"
# (El JSON no aparece porque está en .gitignore)

git log --oneline
# Debe mostrar los 2 commits del laboratorio
```

> **No elimines** el directorio `toolkit/` ni el entorno virtual `.venv/`. Esta estructura será la base de todos los laboratorios siguientes del curso. Solo desactiva el entorno virtual al cerrar la sesión.

---

## 10. Resumen

En este laboratorio completaste los fundamentos del entorno de trabajo que usarás durante todo el curso:

| Tarea completada                                         | Concepto aplicado                              |
|----------------------------------------------------------|------------------------------------------------|
| Estructura de directorios del proyecto creada            | Organización de proyectos Python               |
| Entorno virtual con `venv` configurado y activado        | Aislamiento de dependencias (Lección 1.1)      |
| Repositorio Git inicializado con `.gitignore` correcto   | Control de versiones y seguridad de secretos   |
| Funciones con `try/except` y docstrings implementadas    | Manejo de errores y documentación              |
| Datos del sistema recolectados con módulos estándar      | `os`, `sys`, `socket`, `platform`, `subprocess`|
| Resultados exportados a JSON estructurado                | Serialización de datos con `json.dump`         |
| Patrón `if __name__ == "__main__"` aplicado              | Separación de lógica y punto de entrada        |

### Conceptos clave reforzados

- **Entornos virtuales:** Cada proyecto tiene su propio `.venv/` para evitar conflictos de dependencias entre herramientas.
- **Manejo de errores:** Las funciones de red y sistema siempre usan `try/except` porque estas operaciones pueden fallar por razones externas al código.
- **`.gitignore` desde el inicio:** Los archivos de datos (`data/*.json`) y las claves API (`.env`) nunca deben versionarse — este hábito es crítico en seguridad.
- **Funciones reutilizables:** Cada función hace una sola cosa y retorna un tipo de dato claro — este diseño modular facilitará integrar estos componentes en el toolkit final.

### Próximos pasos

En el **Lab 01-00-02** profundizarás en variables tipadas, funciones avanzadas y manejo de excepciones específicas de red con `socket`. El script de este laboratorio será extendido para incorporar reconocimiento pasivo usando `python-whois` y `dnspython`.

### Recursos adicionales

- [Documentación oficial de `venv`](https://docs.python.org/3/library/venv.html)
- [Documentación del módulo `socket`](https://docs.python.org/3/library/socket.html)
- [Documentación del módulo `platform`](https://docs.python.org/3/library/platform.html)
- [PEP 8 — Guía de estilo oficial de Python](https://peps.python.org/pep-0008/)
- [Referencia de `json.dump` y `json.load`](https://docs.python.org/3/library/json.html)

---
*Lab 01-00-01 — Curso: Hacking Ético con Python | Capítulo 1, Lección 1.1*
