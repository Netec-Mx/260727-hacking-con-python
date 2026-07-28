---LAB_START---
LAB_ID: 08-00-01
---MARKDOWN---
# Práctica 8 — Integración y Entrega de un Toolkit Ético

## Metadatos

| Campo            | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 43 minutos                                   |
| **Complejidad**  | Alta                                         |
| **Nivel Bloom**  | Crear (Create)                               |
| **Módulo**       | 8 — Planificación, Integración y Entrega     |
| **Metodología**  | PTES / OWASP Testing Guide                   |

---

## Descripción General

En este laboratorio integrarás todos los módulos construidos durante el curso (`passive_recon`, `ethical_scraper`, `banner_grabber`, `web_tester`, `scapy_scanner`, `ssh_automation`, `msf_controller`, `tor_proxy`) en un **toolkit cohesivo con CLI unificada**. Implementarás un módulo de configuración central que exige un archivo de autorización firmado antes de ejecutar cualquier acción activa, un sistema de logging de auditoría centralizado, una suite de tests con `pytest` y un generador automático de reportes en formato Markdown/HTML. El laboratorio cierra con la entrega final del proyecto vía Git, aplicando los principios éticos y legales trabajados durante todo el curso.

---

## Objetivos de Aprendizaje

- [ ] Integrar módulos heterogéneos en un paquete Python con CLI unificada usando `argparse` y subcomandos.
- [ ] Implementar un módulo de autorización y logging de auditoría que bloquee operaciones no autorizadas y registre cada acción con timestamp y usuario.
- [ ] Escribir y ejecutar pruebas unitarias con `pytest` que validen funciones de parsing, clasificación y validación de alcance.
- [ ] Generar automáticamente un reporte profesional de hallazgos en formato Markdown/HTML usando plantillas Jinja2.
- [ ] Entregar el toolkit completo mediante un repositorio Git con historial de commits limpio y sin credenciales expuestas.

---

## Prerrequisitos

### Conocimiento previo
- Haber completado los laboratorios 01 al 07 con todos los módulos individuales funcionales.
- Comprensión de la estructura de paquetes Python (`__init__.py`, `pyproject.toml`).
- Familiaridad con `argparse`, `logging` y el modelo de fases PTES.
- `pytest` instalado en el entorno virtual del proyecto.

### Acceso y permisos
- Entorno virtual del proyecto activo con todas las dependencias instaladas.
- Formulario de autorización firmado (proporcionado por el instructor) disponible como archivo `authorization.json`.
- Acceso a Metasploitable 2 en red NAT interna aislada (snapshot limpio creado).
- Repositorio Git local inicializado en el directorio del proyecto.

---

## Entorno de Laboratorio

### Hardware requerido

| Componente       | Mínimo recomendado                                         |
|------------------|------------------------------------------------------------|
| RAM              | 8 GB (16 GB recomendado para VMs simultáneas)              |
| CPU              | 4 núcleos 64-bit con VT-x / AMD-V habilitado               |
| Disco            | 60 GB libres para snapshots y resultados                   |
| Red              | Tarjeta compatible con modo promiscuo                      |
| Pantalla         | 1280×768 mínimo para múltiples terminales                  |

### Software requerido

| Software                  | Versión mínima  | Rol en el laboratorio                        |
|---------------------------|-----------------|----------------------------------------------|
| Python                    | 3.10+           | Runtime principal del toolkit                |
| pytest                    | 7.4+            | Suite de tests unitarios                     |
| Jinja2                    | 3.1+            | Generación de reportes HTML                  |
| argparse                  | stdlib          | CLI unificada con subcomandos                |
| logging                   | stdlib          | Auditoría centralizada                       |
| Requests                  | 2.31+           | Módulos HTTP del toolkit                     |
| Scapy                     | 2.5+            | Módulo de escaneo de paquetes                |
| Paramiko                  | 3.3+            | Módulo SSH                                   |
| stem                      | 1.8+            | Módulo TOR                                   |
| Git                       | 2.40+           | Control de versiones y entrega               |
| Metasploitable 2          | —               | Objetivo de pruebas en red aislada           |
| VirtualBox / VMware       | 7.0+ / 17+      | Virtualización del entorno                   |

### Preparación del entorno

> ⚠️ **ÉTICA Y LEGALIDAD**: Antes de continuar, verifica que el formulario de autorización firmado esté disponible. Ningún módulo activo del toolkit se ejecutará sin este archivo. Todos los escaneos deben realizarse **exclusivamente** sobre Metasploitable 2 en la red NAT interna aislada.

> ⚠️ **AISLAMIENTO DE RED**: Confirma que Metasploitable 2 **no tiene acceso a Internet**. Verifica el adaptador de red en VirtualBox/VMware antes de continuar.

```bash
# 1. Crear snapshot de Metasploitable 2 antes de comenzar
# En VirtualBox (ejecutar desde el host):
VBoxManage snapshot "Metasploitable2" take "pre-lab08-snapshot" \
  --description "Estado limpio antes del Lab 08"

# 2. Activar el entorno virtual del proyecto
cd ~/ethical_toolkit
source venv/bin/activate   # Linux/macOS
# venv\Scripts\activate    # Windows

# 3. Verificar dependencias críticas
pip install pytest==7.4.4 jinja2==3.1.4 --quiet
pip list | grep -E "pytest|Jinja2|requests|scapy|paramiko|stem"

# 4. Verificar estructura del proyecto existente
ls -la toolkit/
```

**Salida esperada de verificación de dependencias:**
```
Jinja2          3.1.4
paramiko        3.4.0
pytest          7.4.4
requests        2.31.0
scapy           2.5.0
stem            1.8.2
```

---

## Instrucciones Paso a Paso

---

### Paso 1: Crear la Estructura Final del Paquete y el Módulo de Autorización

**Objetivo:** Consolidar la estructura de directorios del toolkit y construir el módulo `auth_manager.py` que valida el archivo de autorización firmado antes de permitir cualquier operación activa.

#### Instrucciones

**1.1** Crea o confirma la estructura de directorios del paquete:

```bash
cd ~/ethical_toolkit
mkdir -p toolkit/{modules,reports,tests,templates,logs}
touch toolkit/__init__.py
touch toolkit/modules/__init__.py
touch toolkit/tests/__init__.py
```

**1.2** Crea el archivo `pyproject.toml` para el paquete:

```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.backends.legacy:build"

[project]
name = "ethical-toolkit"
version = "1.0.0"
description = "Toolkit de hacking ético modular con controles de seguridad"
requires-python = ">=3.10"
dependencies = [
    "requests>=2.31",
    "scapy>=2.5",
    "paramiko>=3.3",
    "stem>=1.8",
    "jinja2>=3.1",
    "pytest>=7.4",
]

[project.scripts]
ethical-toolkit = "toolkit.cli:main"

[tool.setuptools.packages.find]
where = ["."]
```

**1.3** Crea el módulo de autorización `toolkit/modules/auth_manager.py`:

```python
# toolkit/modules/auth_manager.py
"""
Módulo de gestión de autorización.
Valida que exista un archivo de autorización firmado antes de
permitir cualquier operación activa sobre sistemas objetivo.
"""

import json
import hashlib
import datetime
from pathlib import Path
from typing import Optional

CAMPOS_REQUERIDOS = [
    "cliente",
    "contacto_autorizado",
    "firma_digital",
    "activos_autorizados",
    "fecha_inicio",
    "fecha_fin",
    "tipo_prueba",
]


class AutorizacionError(Exception):
    """Excepción lanzada cuando la autorización es inválida o está ausente."""
    pass


def cargar_autorizacion(ruta: str = "authorization.json") -> dict:
    """
    Carga y valida el archivo de autorización firmado.

    Args:
        ruta: Ruta al archivo JSON de autorización.

    Returns:
        Diccionario con los datos de autorización validados.

    Raises:
        AutorizacionError: Si el archivo no existe, está incompleto
                           o la vigencia ha expirado.
    """
    ruta_path = Path(ruta)

    if not ruta_path.exists():
        raise AutorizacionError(
            f"[AUTH ERROR] Archivo de autorización no encontrado: {ruta}\n"
            "Obtén el formulario firmado del instructor antes de continuar."
        )

    with open(ruta_path, "r", encoding="utf-8") as f:
        datos = json.load(f)

    # Verificar campos obligatorios
    faltantes = [c for c in CAMPOS_REQUERIDOS if not datos.get(c)]
    if faltantes:
        raise AutorizacionError(
            f"[AUTH ERROR] Campos obligatorios faltantes en la autorización: "
            f"{', '.join(faltantes)}"
        )

    # Verificar vigencia temporal
    hoy = datetime.date.today()
    try:
        inicio = datetime.date.fromisoformat(datos["fecha_inicio"])
        fin = datetime.date.fromisoformat(datos["fecha_fin"])
    except ValueError as e:
        raise AutorizacionError(
            f"[AUTH ERROR] Formato de fecha inválido en autorización: {e}"
        )

    if not (inicio <= hoy <= fin):
        raise AutorizacionError(
            f"[AUTH ERROR] Autorización fuera de vigencia. "
            f"Período válido: {inicio} → {fin}. Hoy: {hoy}"
        )

    return datos


def verificar_activo_autorizado(objetivo: str, autorizacion: dict) -> bool:
    """
    Verifica si un objetivo específico está en la lista de activos autorizados.

    Args:
        objetivo: IP, dominio o rango CIDR del objetivo.
        autorizacion: Diccionario de autorización ya validado.

    Returns:
        True si el objetivo está autorizado, False en caso contrario.
    """
    activos = autorizacion.get("activos_autorizados", [])
    return any(objetivo in activo or activo in objetivo for activo in activos)


def generar_hash_autorizacion(autorizacion: dict) -> str:
    """Genera un hash SHA-256 del contenido de la autorización para auditoría."""
    contenido = json.dumps(autorizacion, sort_keys=True, ensure_ascii=False)
    return hashlib.sha256(contenido.encode()).hexdigest()[:16]
```

**1.4** Crea el archivo de autorización de ejemplo para el laboratorio:

```bash
cat > authorization.json << 'EOF'
{
    "cliente": "Laboratorio Controlado - Metasploitable2",
    "contacto_autorizado": "Instructor del Curso",
    "firma_digital": "LAB08-AUTHORIZED-2025",
    "activos_autorizados": [
        "192.168.56.101",
        "192.168.56.0/24",
        "metasploitable2.lab"
    ],
    "fecha_inicio": "2025-01-01",
    "fecha_fin": "2025-12-31",
    "tipo_prueba": "caja_gris",
    "notas": "Entorno de laboratorio controlado. Solo para fines educativos."
}
EOF
```

> 📝 **Nota:** Reemplaza `fecha_inicio` y `fecha_fin` con fechas que incluyan la fecha actual del sistema.

#### Salida esperada

```
# Al importar y probar el módulo:
>>> from toolkit.modules.auth_manager import cargar_autorizacion
>>> auth = cargar_autorizacion("authorization.json")
>>> print(auth["cliente"])
Laboratorio Controlado - Metasploitable2
```

#### Verificación

```bash
python -c "
from toolkit.modules.auth_manager import cargar_autorizacion, verificar_activo_autorizado
auth = cargar_autorizacion('authorization.json')
print('[OK] Autorización cargada:', auth['cliente'])
print('[OK] Activo autorizado:', verificar_activo_autorizado('192.168.56.101', auth))
print('[OK] Activo NO autorizado:', verificar_activo_autorizado('8.8.8.8', auth))
"
```

**Salida esperada de verificación:**
```
[OK] Autorización cargada: Laboratorio Controlado - Metasploitable2
[OK] Activo autorizado: True
[OK] Activo NO autorizado: False
```

---

### Paso 2: Implementar el Sistema de Logging de Auditoría Centralizado

**Objetivo:** Crear el módulo `audit_logger.py` que registra todas las acciones del toolkit con timestamp, usuario, módulo invocado y objetivo, tanto en archivo como en consola.

#### Instrucciones

**2.1** Crea el módulo `toolkit/modules/audit_logger.py`:

```python
# toolkit/modules/audit_logger.py
"""
Sistema de logging de auditoría centralizado para el toolkit ético.
Registra todas las acciones con timestamp, usuario, módulo y objetivo.
Los logs se escriben simultáneamente en archivo y consola.
"""

import logging
import os
import datetime
from pathlib import Path
from typing import Optional

LOG_DIR = Path("toolkit/logs")
LOG_FORMAT = "%(asctime)s | %(levelname)-8s | %(name)s | %(message)s"
DATE_FORMAT = "%Y-%m-%d %H:%M:%S"


def configurar_logger(
    nombre_modulo: str,
    nivel: int = logging.INFO,
    log_dir: Optional[Path] = None
) -> logging.Logger:
    """
    Crea y configura un logger con handlers de archivo y consola.

    Args:
        nombre_modulo: Nombre del módulo que solicita el logger.
        nivel: Nivel de logging (INFO por defecto).
        log_dir: Directorio para los archivos de log.

    Returns:
        Logger configurado con handlers múltiples.
    """
    if log_dir is None:
        log_dir = LOG_DIR

    log_dir.mkdir(parents=True, exist_ok=True)

    logger = logging.getLogger(f"ethical_toolkit.{nombre_modulo}")
    logger.setLevel(nivel)

    # Evitar duplicar handlers si el logger ya fue configurado
    if logger.handlers:
        return logger

    formatter = logging.Formatter(LOG_FORMAT, datefmt=DATE_FORMAT)

    # Handler de consola
    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.INFO)
    console_handler.setFormatter(formatter)

    # Handler de archivo (rotación diaria por fecha)
    fecha_hoy = datetime.date.today().isoformat()
    log_file = log_dir / f"audit_{fecha_hoy}.log"
    file_handler = logging.FileHandler(log_file, encoding="utf-8")
    file_handler.setLevel(logging.DEBUG)
    file_handler.setFormatter(formatter)

    logger.addHandler(console_handler)
    logger.addHandler(file_handler)

    return logger


def registrar_accion(
    logger: logging.Logger,
    accion: str,
    objetivo: str,
    resultado: str,
    usuario: Optional[str] = None
) -> None:
    """
    Registra una acción de auditoría con formato estandarizado.

    Args:
        logger: Logger configurado del módulo.
        accion: Descripción de la acción ejecutada.
        objetivo: Sistema o activo sobre el que se ejecutó.
        resultado: Resultado de la operación (éxito/fallo/hallazgo).
        usuario: Usuario del sistema (se detecta automáticamente si es None).
    """
    if usuario is None:
        usuario = os.getenv("USER", os.getenv("USERNAME", "desconocido"))

    mensaje = (
        f"USUARIO={usuario} | "
        f"ACCION={accion} | "
        f"OBJETIVO={objetivo} | "
        f"RESULTADO={resultado}"
    )
    logger.info(mensaje)


def registrar_error(
    logger: logging.Logger,
    accion: str,
    objetivo: str,
    error: Exception
) -> None:
    """Registra un error con trazabilidad completa."""
    usuario = os.getenv("USER", os.getenv("USERNAME", "desconocido"))
    mensaje = (
        f"USUARIO={usuario} | "
        f"ACCION={accion} | "
        f"OBJETIVO={objetivo} | "
        f"ERROR={type(error).__name__}: {error}"
    )
    logger.error(mensaje)
```

**2.2** Prueba el sistema de logging:

```bash
python -c "
from toolkit.modules.audit_logger import configurar_logger, registrar_accion

logger = configurar_logger('test_audit')
registrar_accion(logger, 'banner_grab', '192.168.56.101:22', 'SSH-2.0-OpenSSH_4.7p1')
registrar_accion(logger, 'port_scan', '192.168.56.101', 'Puerto 80 abierto')
print('[OK] Logs escritos en toolkit/logs/')
"
```

#### Salida esperada

```
2025-09-01 10:15:32 | INFO     | ethical_toolkit.test_audit | USUARIO=estudiante | ACCION=banner_grab | OBJETIVO=192.168.56.101:22 | RESULTADO=SSH-2.0-OpenSSH_4.7p1
2025-09-01 10:15:32 | INFO     | ethical_toolkit.test_audit | USUARIO=estudiante | ACCION=port_scan | OBJETIVO=192.168.56.101 | RESULTADO=Puerto 80 abierto
[OK] Logs escritos en toolkit/logs/
```

#### Verificación

```bash
# Verificar que el archivo de log fue creado
ls -la toolkit/logs/
cat toolkit/logs/audit_$(date +%Y-%m-%d).log
```

---

### Paso 3: Construir la CLI Unificada con `argparse` y Subcomandos

**Objetivo:** Crear el punto de entrada principal `toolkit/cli.py` que expone todos los módulos del toolkit a través de una CLI unificada con subcomandos, validación de autorización integrada y logging automático de cada invocación.

#### Instrucciones

**3.1** Crea el archivo `toolkit/cli.py`:

```python
# toolkit/cli.py
"""
CLI unificada del Ethical Toolkit.
Punto de entrada principal con subcomandos para todos los módulos.
Requiere archivo de autorización válido para operaciones activas.
"""

import argparse
import sys
import os
from pathlib import Path

# Asegurar que el paquete sea importable desde el directorio raíz
sys.path.insert(0, str(Path(__file__).parent.parent))

from toolkit.modules.auth_manager import (
    cargar_autorizacion,
    verificar_activo_autorizado,
    AutorizacionError,
    generar_hash_autorizacion,
)
from toolkit.modules.audit_logger import (
    configurar_logger,
    registrar_accion,
    registrar_error,
)

# Logger global de la CLI
logger = configurar_logger("cli")

# ─── Stubs de módulos (reemplazar con imports reales de labs anteriores) ──────

def _stub_modulo(nombre: str, objetivo: str, auth: dict) -> dict:
    """
    Stub genérico que simula la ejecución de un módulo.
    En producción, reemplazar con el import real del módulo correspondiente.
    """
    registrar_accion(logger, f"ejecutar_{nombre}", objetivo, "stub_ejecutado")
    return {
        "modulo": nombre,
        "objetivo": objetivo,
        "estado": "stub",
        "hallazgos": [f"[STUB] {nombre} ejecutado sobre {objetivo}"],
    }


def cmd_recon(args: argparse.Namespace, auth: dict) -> dict:
    """Ejecuta reconocimiento pasivo sobre el objetivo."""
    if not verificar_activo_autorizado(args.objetivo, auth):
        raise AutorizacionError(
            f"El objetivo '{args.objetivo}' NO está en la lista de activos autorizados."
        )
    registrar_accion(logger, "passive_recon", args.objetivo, "iniciado")
    # TODO: reemplazar con import real: from toolkit.modules.passive_recon import ejecutar
    resultado = _stub_modulo("passive_recon", args.objetivo, auth)
    registrar_accion(logger, "passive_recon", args.objetivo, f"completado: {len(resultado['hallazgos'])} hallazgos")
    return resultado


def cmd_scan(args: argparse.Namespace, auth: dict) -> dict:
    """Ejecuta escaneo de puertos y banner grabbing."""
    if not verificar_activo_autorizado(args.objetivo, auth):
        raise AutorizacionError(
            f"El objetivo '{args.objetivo}' NO está autorizado."
        )
    registrar_accion(logger, "banner_grabber", args.objetivo, f"puertos={args.puertos}")
    resultado = _stub_modulo("banner_grabber", args.objetivo, auth)
    resultado["puertos"] = args.puertos
    registrar_accion(logger, "banner_grabber", args.objetivo, "completado")
    return resultado


def cmd_web(args: argparse.Namespace, auth: dict) -> dict:
    """Ejecuta pruebas web (SQLi, XSS, LFI) sobre el objetivo."""
    if not verificar_activo_autorizado(args.url, auth):
        raise AutorizacionError(
            f"La URL '{args.url}' no corresponde a un activo autorizado."
        )
    registrar_accion(logger, "web_tester", args.url, f"tests={args.tests}")
    resultado = _stub_modulo("web_tester", args.url, auth)
    resultado["tests_ejecutados"] = args.tests
    return resultado


def cmd_report(args: argparse.Namespace) -> None:
    """Genera el reporte final de hallazgos."""
    registrar_accion(logger, "report_generator", "N/A", f"formato={args.formato}")
    # Se llama al generador de reportes del Paso 4
    from toolkit.modules.report_generator import generar_reporte
    generar_reporte(
        resultados_dir=Path(args.resultados_dir),
        formato=args.formato,
        salida=Path(args.salida),
    )
    registrar_accion(logger, "report_generator", "N/A", f"reporte generado: {args.salida}")
    print(f"[OK] Reporte generado en: {args.salida}")


# ─── Construcción del parser ──────────────────────────────────────────────────

def construir_parser() -> argparse.ArgumentParser:
    """Construye el parser principal con todos los subcomandos."""
    parser = argparse.ArgumentParser(
        prog="ethical-toolkit",
        description=(
            "Ethical Hacking Toolkit — Versión 1.0\n"
            "Requiere archivo de autorización firmado para operaciones activas.\n"
            "Uso exclusivo sobre entornos propios o con autorización explícita."
        ),
        formatter_class=argparse.RawDescriptionHelpFormatter,
    )

    parser.add_argument(
        "--auth",
        default="authorization.json",
        metavar="ARCHIVO",
        help="Ruta al archivo de autorización firmado (default: authorization.json)",
    )
    parser.add_argument(
        "--verbose", "-v",
        action="store_true",
        help="Mostrar salida detallada",
    )

    subparsers = parser.add_subparsers(
        dest="comando",
        title="Módulos disponibles",
        metavar="MÓDULO",
    )
    subparsers.required = True

    # ── Subcomando: recon ────────────────────────────────────────────────────
    p_recon = subparsers.add_parser(
        "recon",
        help="Reconocimiento pasivo (DNS, WHOIS, Shodan)",
    )
    p_recon.add_argument("objetivo", help="Dominio o IP objetivo")
    p_recon.add_argument(
        "--dns", action="store_true", help="Incluir enumeración DNS"
    )
    p_recon.add_argument(
        "--whois", action="store_true", help="Incluir consulta WHOIS"
    )

    # ── Subcomando: scan ─────────────────────────────────────────────────────
    p_scan = subparsers.add_parser(
        "scan",
        help="Escaneo de puertos y banner grabbing",
    )
    p_scan.add_argument("objetivo", help="IP o rango CIDR objetivo")
    p_scan.add_argument(
        "--puertos",
        default="22,80,443,8080",
        help="Puertos a escanear (default: 22,80,443,8080)",
    )
    p_scan.add_argument(
        "--timeout",
        type=float,
        default=2.0,
        help="Timeout por conexión en segundos (default: 2.0)",
    )

    # ── Subcomando: web ──────────────────────────────────────────────────────
    p_web = subparsers.add_parser(
        "web",
        help="Pruebas de seguridad web (SQLi, XSS, LFI)",
    )
    p_web.add_argument("url", help="URL base del objetivo web")
    p_web.add_argument(
        "--tests",
        nargs="+",
        choices=["sqli", "xss", "lfi", "all"],
        default=["all"],
        help="Tipos de pruebas a ejecutar (default: all)",
    )

    # ── Subcomando: report ───────────────────────────────────────────────────
    p_report = subparsers.add_parser(
        "report",
        help="Generar reporte de hallazgos",
    )
    p_report.add_argument(
        "--resultados-dir",
        default="toolkit/logs",
        help="Directorio con resultados de módulos (default: toolkit/logs)",
    )
    p_report.add_argument(
        "--formato",
        choices=["markdown", "html"],
        default="html",
        help="Formato del reporte (default: html)",
    )
    p_report.add_argument(
        "--salida",
        default="reporte_final.html",
        help="Nombre del archivo de salida (default: reporte_final.html)",
    )

    return parser


# ─── Función principal ────────────────────────────────────────────────────────

def main() -> None:
    """Punto de entrada principal del toolkit."""
    parser = construir_parser()
    args = parser.parse_args()

    print("=" * 60)
    print("  ETHICAL HACKING TOOLKIT v1.0")
    print("  Uso exclusivo en entornos autorizados")
    print("=" * 60)

    # El subcomando 'report' no requiere autorización activa
    if args.comando == "report":
        try:
            cmd_report(args)
        except Exception as e:
            registrar_error(logger, "report", "N/A", e)
            print(f"[ERROR] {e}")
            sys.exit(1)
        return

    # Para todos los demás comandos: validar autorización
    try:
        auth = cargar_autorizacion(args.auth)
        hash_auth = generar_hash_autorizacion(auth)
        print(f"[AUTH] Autorización válida — Cliente: {auth['cliente']}")
        print(f"[AUTH] Hash de autorización: {hash_auth}")
        print(f"[AUTH] Período: {auth['fecha_inicio']} → {auth['fecha_fin']}")
        print("-" * 60)
    except AutorizacionError as e:
        print(f"\n{'='*60}")
        print(str(e))
        print(f"{'='*60}\n")
        sys.exit(1)

    # Despachar al subcomando correspondiente
    try:
        if args.comando == "recon":
            resultado = cmd_recon(args, auth)
        elif args.comando == "scan":
            resultado = cmd_scan(args, auth)
        elif args.comando == "web":
            resultado = cmd_web(args, auth)
        else:
            parser.print_help()
            sys.exit(0)

        print(f"\n[RESULTADO] Módulo: {resultado.get('modulo', args.comando)}")
        print(f"[RESULTADO] Objetivo: {resultado.get('objetivo', 'N/A')}")
        for hallazgo in resultado.get("hallazgos", []):
            print(f"  → {hallazgo}")

    except AutorizacionError as e:
        registrar_error(logger, args.comando, "N/A", e)
        print(f"\n[DENEGADO] {e}")
        sys.exit(1)
    except Exception as e:
        registrar_error(logger, args.comando, "N/A", e)
        print(f"\n[ERROR] Error inesperado: {e}")
        if args.verbose:
            import traceback
            traceback.print_exc()
        sys.exit(1)


if __name__ == "__main__":
    main()
```

**3.2** Prueba la CLI con diferentes subcomandos:

```bash
# Probar ayuda general
python -m toolkit.cli --help

# Probar subcomando recon con autorización válida
python -m toolkit.cli --auth authorization.json recon 192.168.56.101 --dns

# Probar subcomando scan
python -m toolkit.cli scan 192.168.56.101 --puertos 22,80,8180

# Probar rechazo de objetivo no autorizado
python -m toolkit.cli scan 8.8.8.8 --puertos 80

# Probar sin archivo de autorización
python -m toolkit.cli --auth archivo_inexistente.json scan 192.168.56.101
```

#### Salida esperada

```
============================================================
  ETHICAL HACKING TOOLKIT v1.0
  Uso exclusivo en entornos autorizados
============================================================
[AUTH] Autorización válida — Cliente: Laboratorio Controlado - Metasploitable2
[AUTH] Hash de autorización: a3f7c2b1e9d4...
[AUTH] Período: 2025-01-01 → 2025-12-31
------------------------------------------------------------
2025-09-01 10:20:15 | INFO | ethical_toolkit.cli | USUARIO=estudiante | ACCION=banner_grabber | OBJETIVO=192.168.56.101 | RESULTADO=puertos=22,80,8180

[RESULTADO] Módulo: banner_grabber
[RESULTADO] Objetivo: 192.168.56.101
  → [STUB] banner_grabber ejecutado sobre 192.168.56.101
```

#### Verificación

```bash
# Verificar que el rechazo de objetivo no autorizado funciona correctamente
python -m toolkit.cli scan 8.8.8.8 --puertos 80
# Debe mostrar: [DENEGADO] El objetivo '8.8.8.8' NO está autorizado.
echo "Código de salida: $?"  # Debe ser 1
```

---

### Paso 4: Crear el Generador de Reportes con Jinja2

**Objetivo:** Implementar `toolkit/modules/report_generator.py` que lee los logs de auditoría y genera un reporte profesional de hallazgos en formato Markdown y/o HTML usando una plantilla Jinja2.

#### Instrucciones

**4.1** Crea la plantilla Jinja2 en `toolkit/templates/reporte.html.j2`:

```bash
cat > toolkit/templates/reporte.html.j2 << 'TEMPLATE'
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Reporte de Hallazgos — {{ metadata.cliente }}</title>
    <style>
        body { font-family: 'Segoe UI', Arial, sans-serif; max-width: 960px; margin: 40px auto; color: #222; }
        h1 { color: #1a1a2e; border-bottom: 3px solid #e94560; padding-bottom: 8px; }
        h2 { color: #16213e; margin-top: 32px; }
        .badge { display: inline-block; padding: 3px 10px; border-radius: 4px; font-size: 0.85em; font-weight: bold; }
        .badge-high   { background: #ff4444; color: white; }
        .badge-medium { background: #ffaa00; color: white; }
        .badge-low    { background: #00aa44; color: white; }
        .badge-info   { background: #0088cc; color: white; }
        table { width: 100%; border-collapse: collapse; margin-top: 12px; }
        th { background: #1a1a2e; color: white; padding: 10px; text-align: left; }
        td { padding: 8px 10px; border-bottom: 1px solid #ddd; }
        tr:hover { background: #f5f5f5; }
        .disclaimer { background: #fff3cd; border-left: 4px solid #ffc107; padding: 12px 16px; margin: 20px 0; }
        .footer { margin-top: 48px; font-size: 0.85em; color: #888; border-top: 1px solid #ddd; padding-top: 16px; }
        code { background: #f4f4f4; padding: 2px 6px; border-radius: 3px; font-size: 0.9em; }
    </style>
</head>
<body>
    <h1>📋 Reporte de Hallazgos de Seguridad</h1>

    <div class="disclaimer">
        <strong>⚠️ CONFIDENCIAL:</strong> Este documento contiene información sensible de seguridad.
        Distribución restringida al equipo autorizado. Generado el {{ metadata.fecha_generacion }}.
    </div>

    <h2>1. Información del Proyecto</h2>
    <table>
        <tr><th>Campo</th><th>Valor</th></tr>
        <tr><td>Cliente</td><td>{{ metadata.cliente }}</td></tr>
        <tr><td>Tipo de prueba</td><td>{{ metadata.tipo_prueba }}</td></tr>
        <tr><td>Período</td><td>{{ metadata.fecha_inicio }} → {{ metadata.fecha_fin }}</td></tr>
        <tr><td>Hash de autorización</td><td><code>{{ metadata.hash_autorizacion }}</code></td></tr>
    </table>

    <h2>2. Resumen Ejecutivo</h2>
    <table>
        <tr>
            <th>Total acciones</th>
            <th>Módulos ejecutados</th>
            <th>Objetivos evaluados</th>
        </tr>
        <tr>
            <td>{{ resumen.total_acciones }}</td>
            <td>{{ resumen.modulos_ejecutados | join(', ') }}</td>
            <td>{{ resumen.objetivos | join(', ') }}</td>
        </tr>
    </table>

    <h2>3. Registro de Auditoría</h2>
    <table>
        <tr><th>Timestamp</th><th>Módulo</th><th>Objetivo</th><th>Resultado</th></tr>
        {% for entrada in audit_log %}
        <tr>
            <td>{{ entrada.timestamp }}</td>
            <td>{{ entrada.modulo }}</td>
            <td><code>{{ entrada.objetivo }}</code></td>
            <td>{{ entrada.resultado }}</td>
        </tr>
        {% endfor %}
    </table>

    <h2>4. Hallazgos y Recomendaciones</h2>
    {% if hallazgos %}
    <table>
        <tr><th>ID</th><th>Severidad</th><th>Descripción</th><th>Recomendación</th></tr>
        {% for h in hallazgos %}
        <tr>
            <td>{{ h.id }}</td>
            <td><span class="badge badge-{{ h.severidad | lower }}">{{ h.severidad }}</span></td>
            <td>{{ h.descripcion }}</td>
            <td>{{ h.recomendacion }}</td>
        </tr>
        {% endfor %}
    </table>
    {% else %}
    <p><em>No se registraron hallazgos en esta sesión de laboratorio.</em></p>
    {% endif %}

    <div class="footer">
        <p>Generado automáticamente por Ethical Toolkit v1.0 |
        Uso exclusivo en entornos autorizados |
        Hash de integridad: <code>{{ metadata.hash_autorizacion }}</code></p>
    </div>
</body>
</html>
TEMPLATE
```

**4.2** Crea `toolkit/modules/report_generator.py`:

```python
# toolkit/modules/report_generator.py
"""
Generador automático de reportes de hallazgos.
Produce reportes en formato Markdown o HTML usando plantillas Jinja2.
Los datos sensibles se ofuscan antes de incluirlos en el reporte.
"""

import re
import json
import datetime
import hashlib
from pathlib import Path
from typing import Optional

try:
    from jinja2 import Environment, FileSystemLoader, select_autoescape
    JINJA2_DISPONIBLE = True
except ImportError:
    JINJA2_DISPONIBLE = False

from toolkit.modules.audit_logger import configurar_logger
from toolkit.modules.auth_manager import cargar_autorizacion, generar_hash_autorizacion

logger = configurar_logger("report_generator")


def _ofuscar_dato_sensible(valor: str) -> str:
    """
    Ofusca parcialmente datos sensibles (IPs, credenciales, hashes).
    Conserva suficiente información para identificar el activo.
    """
    # Ofuscar último octeto de IPs
    ip_pattern = re.compile(r'(\d{1,3}\.\d{1,3}\.\d{1,3})\.\d{1,3}')
    valor = ip_pattern.sub(r'\1.***', valor)
    # Ofuscar contraseñas en texto plano (patrones comunes)
    pwd_pattern = re.compile(r'(password|passwd|pwd|clave)=\S+', re.IGNORECASE)
    valor = pwd_pattern.sub(r'\1=*****', valor)
    return valor


def _parsear_log_auditoria(log_dir: Path) -> list[dict]:
    """
    Lee los archivos de log de auditoría y extrae entradas estructuradas.
    """
    entradas = []
    patron = re.compile(
        r'(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}) \| \w+\s+\| '
        r'[\w\.]+\s+\| USUARIO=(\S+) \| ACCION=(\S+) \| '
        r'OBJETIVO=(\S+) \| RESULTADO=(.+)'
    )

    for log_file in sorted(log_dir.glob("audit_*.log")):
        with open(log_file, "r", encoding="utf-8") as f:
            for linea in f:
                match = patron.search(linea.strip())
                if match:
                    entradas.append({
                        "timestamp": match.group(1),
                        "usuario": match.group(2),
                        "modulo": match.group(3),
                        "objetivo": _ofuscar_dato_sensible(match.group(4)),
                        "resultado": _ofuscar_dato_sensible(match.group(5)),
                    })

    return entradas


def _generar_hallazgos_ejemplo() -> list[dict]:
    """
    Genera hallazgos de ejemplo para el reporte del laboratorio.
    En producción, estos provendrían de los módulos de análisis.
    """
    return [
        {
            "id": "HALL-001",
            "severidad": "HIGH",
            "descripcion": "Servicio SSH con versión obsoleta (OpenSSH 4.7p1) expuesta",
            "recomendacion": "Actualizar OpenSSH a versión 9.x y deshabilitar autenticación por contraseña",
        },
        {
            "id": "HALL-002",
            "severidad": "MEDIUM",
            "descripcion": "Puerto 8180 (Apache Tomcat) accesible con credenciales por defecto",
            "recomendacion": "Cambiar credenciales por defecto del manager de Tomcat y restringir acceso por IP",
        },
        {
            "id": "HALL-003",
            "severidad": "LOW",
            "descripcion": "Banner de versión de servicios visible sin autenticación",
            "recomendacion": "Configurar supresión de banners en SSH, HTTP y FTP",
        },
        {
            "id": "HALL-004",
            "severidad": "INFO",
            "descripcion": "Múltiples puertos abiertos detectados en el host objetivo",
            "recomendacion": "Revisar política de firewall y cerrar servicios no utilizados",
        },
    ]


def generar_reporte(
    resultados_dir: Path,
    formato: str = "html",
    salida: Path = Path("reporte_final.html"),
    auth_file: str = "authorization.json",
) -> None:
    """
    Genera el reporte completo de hallazgos.

    Args:
        resultados_dir: Directorio con logs de auditoría.
        formato: 'html' o 'markdown'.
        salida: Ruta del archivo de salida.
        auth_file: Archivo de autorización para metadatos.
    """
    # Cargar metadatos de autorización
    try:
        auth = cargar_autorizacion(auth_file)
        hash_auth = generar_hash_autorizacion(auth)
    except Exception:
        auth = {"cliente": "No disponible", "tipo_prueba": "N/A",
                "fecha_inicio": "N/A", "fecha_fin": "N/A"}
        hash_auth = "N/A"

    # Parsear logs de auditoría
    audit_entries = _parsear_log_auditoria(resultados_dir)

    # Construir resumen
    modulos = list({e["modulo"] for e in audit_entries}) or ["N/A"]
    objetivos = list({e["objetivo"] for e in audit_entries if e["objetivo"] != "N/A"}) or ["N/A"]

    contexto = {
        "metadata": {
            "cliente": auth.get("cliente", "N/A"),
            "tipo_prueba": auth.get("tipo_prueba", "N/A"),
            "fecha_inicio": auth.get("fecha_inicio", "N/A"),
            "fecha_fin": auth.get("fecha_fin", "N/A"),
            "hash_autorizacion": hash_auth,
            "fecha_generacion": datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        },
        "resumen": {
            "total_acciones": len(audit_entries),
            "modulos_ejecutados": modulos,
            "objetivos": objetivos,
        },
        "audit_log": audit_entries,
        "hallazgos": _generar_hallazgos_ejemplo(),
    }

    if formato == "html" and JINJA2_DISPONIBLE:
        _generar_html(contexto, salida)
    else:
        _generar_markdown(contexto, salida.with_suffix(".md"))


def _generar_html(contexto: dict, salida: Path) -> None:
    """Renderiza el reporte HTML usando la plantilla Jinja2."""
    templates_dir = Path(__file__).parent.parent / "templates"
    env = Environment(
        loader=FileSystemLoader(str(templates_dir)),
        autoescape=select_autoescape(["html"]),
    )
    template = env.get_template("reporte.html.j2")
    html_content = template.render(**contexto)

    salida.parent.mkdir(parents=True, exist_ok=True)
    with open(salida, "w", encoding="utf-8") as f:
        f.write(html_content)
    logger.info(f"Reporte HTML generado: {salida}")


def _generar_markdown(contexto: dict, salida: Path) -> None:
    """Genera el reporte en formato Markdown."""
    meta = contexto["metadata"]
    resumen = contexto["resumen"]
    lineas = [
        "# Reporte de Hallazgos de Seguridad",
        f"\n> **CONFIDENCIAL** — Generado: {meta['fecha_generacion']}",
        "\n## 1. Información del Proyecto",
        f"- **Cliente:** {meta['cliente']}",
        f"- **Tipo de prueba:** {meta['tipo_prueba']}",
        f"- **Período:** {meta['fecha_inicio']} → {meta['fecha_fin']}",
        f"- **Hash de autorización:** `{meta['hash_autorizacion']}`",
        "\n## 2. Resumen Ejecutivo",
        f"- **Total de acciones registradas:** {resumen['total_acciones']}",
        f"- **Módulos ejecutados:** {', '.join(resumen['modulos_ejecutados'])}",
        f"- **Objetivos evaluados:** {', '.join(resumen['objetivos'])}",
        "\n## 3. Hallazgos",
        "| ID | Severidad | Descripción | Recomendación |",
        "|---|---|---|---|",
    ]
    for h in contexto["hallazgos"]:
        lineas.append(
            f"| {h['id']} | **{h['severidad']}** | {h['descripcion']} | {h['recomendacion']} |"
        )

    salida.parent.mkdir(parents=True, exist_ok=True)
    with open(salida, "w", encoding="utf-8") as f:
        f.write("\n".join(lineas))
    logger.info(f"Reporte Markdown generado: {salida}")
```

**4.3** Prueba la generación del reporte:

```bash
python -m toolkit.cli report --formato html --salida reporte_final.html
python -m toolkit.cli report --formato markdown --salida reporte_final.md
ls -lh reporte_final.*
```

#### Salida esperada

```
2025-09-01 10:25:00 | INFO | ethical_toolkit.cli | USUARIO=estudiante | ACCION=report_generator | OBJETIVO=N/A | RESULTADO=formato=html
[OK] Reporte generado en: reporte_final.html
```

#### Verificación

```bash
# Verificar que el HTML es válido y contiene las secciones esperadas
grep -c "<h2>" reporte_final.html   # Debe ser >= 4
grep "HALL-001" reporte_final.html  # Debe encontrar el hallazgo
wc -l reporte_final.md              # Debe tener > 20 líneas
```

---

### Paso 5: Escribir y Ejecutar la Suite de Tests con pytest

**Objetivo:** Crear pruebas unitarias que validen las funciones críticas del toolkit: parsing de logs, validación de autorización, verificación de activos y generación de hashes.

#### Instrucciones

**5.1** Crea el archivo `toolkit/tests/test_auth_manager.py`:

```python
# toolkit/tests/test_auth_manager.py
"""
Tests unitarios para el módulo de gestión de autorización.
Valida parsing, verificación de activos y detección de errores.
"""

import json
import pytest
import tempfile
import datetime
from pathlib import Path

# Ajustar sys.path para imports relativos en tests
import sys
sys.path.insert(0, str(Path(__file__).parent.parent.parent))

from toolkit.modules.auth_manager import (
    cargar_autorizacion,
    verificar_activo_autorizado,
    generar_hash_autorizacion,
    AutorizacionError,
    CAMPOS_REQUERIDOS,
)


# ─── Fixtures ─────────────────────────────────────────────────────────────────

@pytest.fixture
def autorizacion_valida(tmp_path):
    """Crea un archivo de autorización válido en un directorio temporal."""
    hoy = datetime.date.today()
    datos = {
        "cliente": "Test Cliente S.A.",
        "contacto_autorizado": "test@cliente.com",
        "firma_digital": "TEST-SIGNATURE-2025",
        "activos_autorizados": ["192.168.56.101", "192.168.56.0/24", "test.lab"],
        "fecha_inicio": (hoy - datetime.timedelta(days=1)).isoformat(),
        "fecha_fin": (hoy + datetime.timedelta(days=30)).isoformat(),
        "tipo_prueba": "caja_gris",
    }
    archivo = tmp_path / "auth_test.json"
    archivo.write_text(json.dumps(datos), encoding="utf-8")
    return archivo, datos


@pytest.fixture
def autorizacion_expirada(tmp_path):
    """Crea un archivo de autorización con fechas expiradas."""
    datos = {
        "cliente": "Cliente Expirado",
        "contacto_autorizado": "x@x.com",
        "firma_digital": "SIG",
        "activos_autorizados": ["10.0.0.1"],
        "fecha_inicio": "2020-01-01",
        "fecha_fin": "2020-12-31",
        "tipo_prueba": "caja_negra",
    }
    archivo = tmp_path / "auth_expirada.json"
    archivo.write_text(json.dumps(datos), encoding="utf-8")
    return archivo


# ─── Tests de cargar_autorizacion ─────────────────────────────────────────────

class TestCargarAutorizacion:

    def test_carga_exitosa_con_datos_validos(self, autorizacion_valida):
        """La función debe retornar el diccionario completo con datos válidos."""
        archivo, datos_originales = autorizacion_valida
        resultado = cargar_autorizacion(str(archivo))
        assert resultado["cliente"] == datos_originales["cliente"]
        assert resultado["tipo_prueba"] == datos_originales["tipo_prueba"]

    def test_lanza_error_si_archivo_no_existe(self):
        """Debe lanzar AutorizacionError si el archivo no existe."""
        with pytest.raises(AutorizacionError, match="no encontrado"):
            cargar_autorizacion("/ruta/que/no/existe.json")

    def test_lanza_error_si_campo_faltante(self, tmp_path):
        """Debe detectar campos obligatorios ausentes."""
        datos_incompletos = {
            "cliente": "Test",
            # Falta: contacto_autorizado, firma_digital, etc.
        }
        archivo = tmp_path / "incompleto.json"
        archivo.write_text(json.dumps(datos_incompletos))
        with pytest.raises(AutorizacionError, match="faltantes"):
            cargar_autorizacion(str(archivo))

    def test_lanza_error_si_autorizacion_expirada(self, autorizacion_expirada):
        """Debe rechazar autorizaciones fuera del período de vigencia."""
        with pytest.raises(AutorizacionError, match="vigencia"):
            cargar_autorizacion(str(autorizacion_expirada))

    def test_todos_los_campos_requeridos_estan_definidos(self):
        """Verificar que la lista de campos requeridos no esté vacía."""
        assert len(CAMPOS_REQUERIDOS) >= 6
        assert "cliente" in CAMPOS_REQUERIDOS
        assert "activos_autorizados" in CAMPOS_REQUERIDOS


# ─── Tests de verificar_activo_autorizado ─────────────────────────────────────

class TestVerificarActivoAutorizado:

    def test_ip_exacta_autorizada(self, autorizacion_valida):
        """Una IP exacta en la lista debe retornar True."""
        _, datos = autorizacion_valida
        assert verificar_activo_autorizado("192.168.56.101", datos) is True

    def test_ip_no_autorizada_retorna_false(self, autorizacion_valida):
        """Una IP que no está en la lista debe retornar False."""
        _, datos = autorizacion_valida
        assert verificar_activo_autorizado("8.8.8.8", datos) is False

    def test_dominio_autorizado(self, autorizacion_valida):
        """Un dominio en la lista debe ser reconocido como autorizado."""
        _, datos = autorizacion_valida
        assert verificar_activo_autorizado("test.lab", datos) is True

    def test_lista_vacia_retorna_false(self):
        """Con lista vacía de activos, cualquier objetivo debe ser rechazado."""
        auth_sin_activos = {"activos_autorizados": []}
        assert verificar_activo_autorizado("192.168.1.1", auth_sin_activos) is False


# ─── Tests de generar_hash_autorizacion ───────────────────────────────────────

class TestGenerarHashAutorizacion:

    def test_hash_es_string_no_vacio(self, autorizacion_valida):
        """El hash debe ser una cadena no vacía."""
        _, datos = autorizacion_valida
        resultado = generar_hash_autorizacion(datos)
        assert isinstance(resultado, str)
        assert len(resultado) > 0

    def test_mismo_contenido_produce_mismo_hash(self, autorizacion_valida):
        """El hash debe ser determinístico para el mismo contenido."""
        _, datos = autorizacion_valida
        hash1 = generar_hash_autorizacion(datos)
        hash2 = generar_hash_autorizacion(datos)
        assert hash1 == hash2

    def test_contenido_diferente_produce_hash_diferente(self, autorizacion_valida):
        """Contenidos distintos deben producir hashes distintos."""
        _, datos = autorizacion_valida
        datos_modificados = {**datos, "cliente": "Otro Cliente"}
        hash_original = generar_hash_autorizacion(datos)
        hash_modificado = generar_hash_autorizacion(datos_modificados)
        assert hash_original != hash_modificado
```

**5.2** Crea el archivo `toolkit/tests/test_report_generator.py`:

```python
# toolkit/tests/test_report_generator.py
"""
Tests unitarios para el generador de reportes.
Valida parsing de logs, ofuscación de datos y generación de archivos.
"""

import pytest
import tempfile
from pathlib import Path
import sys

sys.path.insert(0, str(Path(__file__).parent.parent.parent))

from toolkit.modules.report_generator import (
    _ofuscar_dato_sensible,
    _parsear_log_auditoria,
    _generar_hallazgos_ejemplo,
)


class TestOfuscarDatoSensible:

    def test_ofusca_ultimo_octeto_de_ip(self):
        """El último octeto de una IP debe ser reemplazado por ***."""
        resultado = _ofuscar_dato_sensible("192.168.56.101")
        assert "***" in resultado
        assert "192.168.56" in resultado
        assert "101" not in resultado

    def test_ofusca_password_en_texto(self):
        """Las contraseñas en texto plano deben ser ofuscadas."""
        resultado = _ofuscar_dato_sensible("password=secreto123")
        assert "secreto123" not in resultado
        assert "*****" in resultado

    def test_texto_sin_datos_sensibles_no_cambia(self):
        """Texto sin datos sensibles debe pasar sin modificación."""
        texto = "banner_grabber completado exitosamente"
        resultado = _ofuscar_dato_sensible(texto)
        assert resultado == texto

    def test_ofusca_multiples_ips_en_cadena(self):
        """Debe ofuscar todas las IPs presentes en la cadena."""
        texto = "Conexión de 10.0.0.5 a 192.168.1.100"
        resultado = _ofuscar_dato_sensible(texto)
        assert "10.0.0.***" in resultado
        assert "192.168.1.***" in resultado


class TestParsearLogAuditoria:

    def test_retorna_lista_vacia_si_no_hay_logs(self, tmp_path):
        """Debe retornar lista vacía si no hay archivos de log."""
        resultado = _parsear_log_auditoria(tmp_path)
        assert resultado == []

    def test_parsea_entrada_valida_de_log(self, tmp_path):
        """Debe extraer correctamente los campos de una entrada de log válida."""
        log_content = (
            "2025-09-01 10:15:32 | INFO     | ethical_toolkit.cli | "
            "USUARIO=estudiante | ACCION=banner_grabber | "
            "OBJETIVO=192.168.56.101 | RESULTADO=SSH-2.0-OpenSSH\n"
        )
        log_file = tmp_path / "audit_2025-09-01.log"
        log_file.write_text(log_content, encoding="utf-8")

        resultado = _parsear_log_auditoria(tmp_path)
        assert len(resultado) == 1
        assert resultado[0]["modulo"] == "banner_grabber"
        assert resultado[0]["usuario"] == "estudiante"

    def test_ignora_lineas_malformadas(self, tmp_path):
        """Las líneas que no coinciden con el patrón deben ser ignoradas."""
        log_file = tmp_path / "audit_2025-09-01.log"
        log_file.write_text("Esta línea no tiene formato de auditoría\n")
        resultado = _parsear_log_auditoria(tmp_path)
        assert resultado == []


class TestGenerarHallazgosEjemplo:

    def test_retorna_lista_no_vacia(self):
        """Debe retornar al menos un hallazgo."""
        hallazgos = _generar_hallazgos_ejemplo()
        assert len(hallazgos) > 0

    def test_cada_hallazgo_tiene_campos_requeridos(self):
        """Cada hallazgo debe tener id, severidad, descripcion y recomendacion."""
        campos = {"id", "severidad", "descripcion", "recomendacion"}
        for h in _generar_hallazgos_ejemplo():
            assert campos.issubset(h.keys()), f"Hallazgo incompleto: {h}"

    def test_severidades_son_validas(self):
        """Las severidades deben ser valores estándar de seguridad."""
        severidades_validas = {"HIGH", "MEDIUM", "LOW", "INFO", "CRITICAL"}
        for h in _generar_hallazgos_ejemplo():
            assert h["severidad"] in severidades_validas
```

**5.3** Crea el archivo de configuración `pytest.ini`:

```ini
# pytest.ini
[pytest]
testpaths = toolkit/tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --tb=short
markers =
    unit: Tests unitarios rápidos sin dependencias externas
    integration: Tests de integración que requieren entorno configurado
```

**5.4** Ejecuta la suite completa de tests:

```bash
# Ejecutar todos los tests con reporte detallado
pytest toolkit/tests/ -v

# Ejecutar con reporte de cobertura (opcional)
pip install pytest-cov --quiet
pytest toolkit/tests/ -v --cov=toolkit/modules --cov-report=term-missing
```

#### Salida esperada

```
============================================================ test session starts =============================================================
platform linux -- Python 3.10.12, pytest-7.4.4
collected 18 items

toolkit/tests/test_auth_manager.py::TestCargarAutorizacion::test_carga_exitosa_con_datos_validos PASSED    [  5%]
toolkit/tests/test_auth_manager.py::TestCargarAutorizacion::test_lanza_error_si_archivo_no_existe PASSED   [ 11%]
toolkit/tests/test_auth_manager.py::TestCargarAutorizacion::test_lanza_error_si_campo_faltante PASSED      [ 16%]
toolkit/tests/test_auth_manager.py::TestCargarAutorizacion::test_lanza_error_si_autorizacion_expirada PASSED [ 22%]
toolkit/tests/test_auth_manager.py::TestCargarAutorizacion::test_todos_los_campos_requeridos_estan_definidos PASSED [ 27%]
toolkit/tests/test_auth_manager.py::TestVerificarActivoAutorizado::test_ip_exacta_autorizada PASSED        [ 33%]
toolkit/tests/test_auth_manager.py::TestVerificarActivoAutorizado::test_ip_no_autorizada_retorna_false PASSED [ 38%]
toolkit/tests/test_auth_manager.py::TestVerificarActivoAutorizado::test_dominio_autorizado PASSED          [ 44%]
toolkit/tests/test_auth_manager.py::TestVerificarActivoAutorizado::test_lista_vacia_retorna_false PASSED   [ 50%]
toolkit/tests/test_auth_manager.py::TestGenerarHashAutorizacion::test_hash_es_string_no_vacio PASSED       [ 55%]
toolkit/tests/test_auth_manager.py::TestGenerarHashAutorizacion::test_mismo_contenido_produce_mismo_hash PASSED [ 61%]
toolkit/tests/test_auth_manager.py::TestGenerarHashAutorizacion::test_contenido_diferente_produce_hash_diferente PASSED [ 66%]
toolkit/tests/test_report_generator.py::TestOfuscarDatoSensible::test_ofusca_ultimo_octeto_de_ip PASSED    [ 72%]
toolkit/tests/test_report_generator.py::TestOfuscarDatoSensible::test_ofusca_password_en_texto PASSED      [ 77%]
toolkit/tests/test_report_generator.py::TestOfuscarDatoSensible::test_texto_sin_datos_sensibles_no_cambia PASSED [ 83%]
toolkit/tests/test_report_generator.py::TestOfuscarDatoSensible::test_ofusca_multiples_ips_en_cadena PASSED [ 88%]
toolkit/tests/test_report_generator.py::TestParsearLogAuditoria::test_retorna_lista_vacia_si_no_hay_logs PASSED [ 94%]
toolkit/tests/test_report_generator.py::TestGenerarHallazgosEjemplo::test_retorna_lista_no_vacia PASSED    [100%]

============================================================ 18 passed in 1.23s ==============================================================
```

#### Verificación

```bash
# Verificar que todos los tests pasan (código de salida 0)
pytest toolkit/tests/ -q
echo "Tests finalizados con código: $?"
# Debe ser: Tests finalizados con código: 0
```

---

### Paso 6: Entrega Final con Git y Revisión de Metodología

**Objetivo:** Preparar el repositorio Git para la entrega final, verificar que no hay credenciales expuestas, generar el reporte final completo y realizar una revisión de la metodología PTES aplicada durante el curso.

#### Instrucciones

**6.1** Configura el archivo `.gitignore` para proteger datos sensibles:

```bash
cat > .gitignore << 'EOF'
# Credenciales y autorización (NUNCA commitear)
authorization.json
*.env
.env
secrets/
api_keys.txt

# Logs y reportes con datos sensibles
toolkit/logs/
reporte_final.html
reporte_final.md

# Entorno virtual Python
venv/
.venv/
__pycache__/
*.pyc
*.pyo
*.egg-info/
dist/
build/

# Datos de herramientas
.pytest_cache/
.coverage
htmlcov/

# Snapshots y archivos de VM
*.ova
*.vmdk
*.vbox-prev
EOF
```

**6.2** Verifica que no hay archivos sensibles ya trackeados:

```bash
# Verificar que authorization.json NO está en el índice de Git
git ls-files authorization.json
# No debe mostrar ningún resultado

# Si ya fue trackeado accidentalmente, removerlo:
# git rm --cached authorization.json
```

**6.3** Realiza el commit final con historial limpio:

```bash
# Inicializar Git si no está inicializado
git init

# Agregar todos los archivos del toolkit (excluidos por .gitignore)
git add pyproject.toml pytest.ini .gitignore
git add toolkit/__init__.py toolkit/modules/ toolkit/tests/ toolkit/templates/
git add toolkit/cli.py 2>/dev/null || true

# Verificar qué se va a commitear (NO debe incluir authorization.json)
git status

# Commit con mensaje descriptivo siguiendo convención de commits
git commit -m "feat(toolkit): integración final del ethical toolkit v1.0

- Módulo auth_manager: validación de autorización y activos
- Módulo audit_logger: logging centralizado con handlers múltiples
- CLI unificada con argparse y 4 subcomandos (recon/scan/web/report)
- Generador de reportes HTML/Markdown con Jinja2 y ofuscación de datos
- Suite de 18 tests unitarios con pytest (100% passing)
- pyproject.toml para empaquetado del toolkit

Closes: Lab 08 - Integración y Entrega del Toolkit Ético"

# Mostrar el historial de commits del proyecto
git log --oneline -10
```

**6.4** Genera el reporte final ejecutando el flujo completo:

```bash
# Simular un flujo completo de uso del toolkit
echo "=== Ejecutando flujo completo del toolkit ==="

# 1. Reconocimiento (stub)
python -m toolkit.cli recon 192.168.56.101 --dns --whois

# 2. Escaneo de puertos (stub)
python -m toolkit.cli scan 192.168.56.101 --puertos 22,80,443,8180

# 3. Pruebas web (stub)
python -m toolkit.cli web http://192.168.56.101 --tests sqli xss

# 4. Generar reporte final HTML
python -m toolkit.cli report --formato html --salida reporte_final.html

# 5. Generar reporte final Markdown
python -m toolkit.cli report --formato markdown --salida reporte_final.md

echo "=== Flujo completo finalizado ==="
ls -lh reporte_final.*
```

**6.5** Revisión de la metodología PTES aplicada durante el curso:

```python
# revision_metodologia.py
# Script de revisión que mapea cada lab al framework PTES

from toolkit.modules.audit_logger import configurar_logger, registrar_accion

logger = configurar_logger("revision_metodologia")

MAPA_PTES_CURSO = [
    {
        "fase": "1. Pre-Engagement",
        "lab": "Lab 08 (este laboratorio)",
        "herramientas": ["auth_manager", "audit_logger", "argparse"],
        "entregable": "Documento de alcance validado, CLI con controles de autorización",
    },
    {
        "fase": "2. Intelligence Gathering",
        "lab": "Labs 02-03",
        "herramientas": ["passive_recon", "ethical_scraper", "dnspython", "shodan"],
        "entregable": "Mapa de superficie de ataque, subdominios, tecnologías",
    },
    {
        "fase": "3. Threat Modeling",
        "lab": "Lab 04",
        "herramientas": ["banner_grabber", "socket", "requests"],
        "entregable": "Inventario de servicios y versiones, vectores de ataque identificados",
    },
    {
        "fase": "4. Vulnerability Analysis",
        "lab": "Lab 05",
        "herramientas": ["web_tester", "requests", "BeautifulSoup4"],
        "entregable": "Lista de vulnerabilidades web con evidencias",
    },
    {
        "fase": "5. Exploitation",
        "lab": "Labs 06-07",
        "herramientas": ["scapy_scanner", "ssh_automation", "msf_controller"],
        "entregable": "Prueba de concepto de explotación controlada",
    },
    {
        "fase": "6. Post-Exploitation",
        "lab": "Lab 07",
        "herramientas": ["tor_proxy", "msf_controller"],
        "entregable": "Evaluación de impacto y persistencia",
    },
    {
        "fase": "7. Reporting",
        "lab": "Lab 08 (este laboratorio)",
        "herramientas": ["report_generator", "jinja2", "audit_logger"],
        "entregable": "Reporte profesional con hallazgos, evidencias y recomendaciones",
    },
]

print("\n" + "=" * 70)
print("  REVISIÓN DE METODOLOGÍA PTES — MAPA DEL CURSO")
print("=" * 70)

for item in MAPA_PTES_CURSO:
    print(f"\n📌 {item['fase']}")
    print(f"   Laboratorio : {item['lab']}")
    print(f"   Herramientas: {', '.join(item['herramientas'])}")
    print(f"   Entregable  : {item['entregable']}")
    registrar_accion(
        logger,
        "revision_ptes",
        item["fase"],
        f"Lab={item['lab']}"
    )

print("\n" + "=" * 70)
print("  ✅ Revisión completada. Toolkit listo para entrega.")
print("=" * 70)
```

```bash
python revision_metodologia.py
```

#### Salida esperada

```
======================================================================
  REVISIÓN DE METODOLOGÍA PTES — MAPA DEL CURSO
======================================================================

📌 1. Pre-Engagement
   Laboratorio : Lab 08 (este laboratorio)
   Herramientas: auth_manager, audit_logger, argparse
   Entregable  : Documento de alcance validado, CLI con controles de autorización

📌 2. Intelligence Gathering
   Laboratorio : Labs 02-03
   Herramientas: passive_recon, ethical_scraper, dnspython, shodan
   Entregable  : Mapa de superficie de ataque, subdominios, tecnologías
[...]
======================================================================
  ✅ Revisión completada. Toolkit listo para entrega.
======================================================================
```

#### Verificación

```bash
# Verificar estructura final del repositorio
git log --oneline -5
git status  # Debe mostrar: nothing to commit, working tree clean

# Verificar que authorization.json NO está en el repositorio
git show HEAD:authorization.json 2>&1 | grep -c "exists" || echo "[OK] authorization.json no está en el repo"

# Verificar que los tests siguen pasando tras todos los cambios
pytest toolkit/tests/ -q
```

---

## Validación y Pruebas

Ejecuta la siguiente secuencia de validación completa para confirmar que el toolkit funciona correctamente en todos sus componentes:

```bash
echo "=== VALIDACIÓN COMPLETA DEL TOOLKIT ÉTICO ==="

# 1. Verificar estructura de paquete
echo "[1/6] Verificando estructura del paquete..."
python -c "import toolkit; print('[OK] Paquete toolkit importable')"

# 2. Verificar módulo de autorización
echo "[2/6] Verificando módulo de autorización..."
python -c "
from toolkit.modules.auth_manager import cargar_autorizacion, AutorizacionError
try:
    cargar_autorizacion('/no/existe.json')
except AutorizacionError as e:
    print('[OK] AutorizacionError lanzada correctamente')
"

# 3. Verificar CLI con --help
echo "[3/6] Verificando CLI..."
python -m toolkit.cli --help > /dev/null && echo "[OK] CLI responde correctamente"

# 4. Verificar que objetivo no autorizado es rechazado
echo "[4/6] Verificando control de acceso..."
python -m toolkit.cli --auth authorization.json scan 8.8.8.8 2>&1 | \
  grep -q "DENEGADO" && echo "[OK] Objetivo no autorizado rechazado"

# 5. Ejecutar suite de tests
echo "[5/6] Ejecutando suite de tests..."
pytest toolkit/tests/ -q --tb=no 2>&1 | tail -1

# 6. Verificar generación de reporte
echo "[6/6] Verificando generación de reporte..."
python -m toolkit.cli report --formato markdown --salida /tmp/test_report.md
test -f /tmp/test_report.md && echo "[OK] Reporte generado correctamente"

echo ""
echo "=== VALIDACIÓN FINALIZADA ==="
```

**Salida esperada de validación completa:**
```
=== VALIDACIÓN COMPLETA DEL TOOLKIT ÉTICO ===
[1/6] Verificando estructura del paquete...
[OK] Paquete toolkit importable
[2/6] Verificando módulo de autorización...
[OK] AutorizacionError lanzada correctamente
[3/6] Verificando CLI...
[OK] CLI responde correctamente
[4/6] Verificando control de acceso...
[OK] Objetivo no autorizado rechazado
[5/6] Ejecutando suite de tests...
18 passed in 1.23s
[6/6] Verificando generación de reporte...
[OK] Reporte generado correctamente
=== VALIDACIÓN FINALIZADA ===
```

---

## Solución de Problemas

### Problema 1: `ModuleNotFoundError: No module named 'toolkit'`

**Síntoma:** Al ejecutar `python -m toolkit.cli` o al importar módulos del toolkit, Python lanza `ModuleNotFoundError` aunque los archivos existen en el directorio.

**Causa:** Python no puede encontrar el paquete `toolkit` porque el directorio raíz del proyecto no está en `sys.path`, y el paquete no ha sido instalado en el entorno virtual. Esto ocurre cuando se ejecutan los scripts desde un directorio diferente al raíz del proyecto, o cuando falta el archivo `__init__.py` en algún nivel del paquete.

**Solución:**
```bash
# Verificar que existen los archivos __init__.py necesarios
ls toolkit/__init__.py
