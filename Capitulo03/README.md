# Práctica 3 — Automatizar consultas respetuosas y almacenar resultados

## 1. Metadatos

| Campo            | Valor                                                              |
|------------------|--------------------------------------------------------------------|
| **Duración**     | 46 minutos                                                         |
| **Complejidad**  | Difícil                                                            |
| **Nivel Bloom**  | Crear                                                              |
| **Módulo**       | 3 — Consultas éticas, concurrencia y persistencia                  |
| **Laboratorio**  | 03-00-01                                                           |

---

## 2. Descripción General

En este laboratorio extenderás el trabajo del Lab 02-00-01 para construir un **sistema de recolección de datos ético, robusto y persistente**. Implementarás la clase `PaginatedQuery` vista en la Lección 3.1, la enriquecerás con un sistema de reintentos con backoff exponencial, añadirás concurrencia controlada mediante `ThreadPoolExecutor` (máximo 3 hilos) y almacenarás los resultados en una base de datos SQLite con esquema normalizado usando SQLAlchemy 2.0. Todo el sistema estará parametrizado mediante un archivo de configuración YAML que centraliza delays, reintentos y límites de peticiones, garantizando un comportamiento ético y auditable.

Las APIs objetivo son **HackerNews (Algolia)** y **GitHub API pública** — ambas de acceso libre y sin restricciones legales para consultas automatizadas moderadas.

---

## 3. Objetivos de Aprendizaje

- [ ] Diseñar rutinas de consulta con paginación automática (page, offset, cursor, Link) que gestionen correctamente los límites de resultados por página de APIs públicas.
- [ ] Implementar concurrencia controlada con `ThreadPoolExecutor` aplicando delays éticos y backoff exponencial entre peticiones.
- [ ] Almacenar y normalizar resultados en SQLite usando el ORM de SQLAlchemy 2.0 con un esquema relacional apropiado.
- [ ] Crear un módulo de configuración YAML que centralice parámetros de delay, reintentos, límites y credenciales de forma segura.
- [ ] Integrar el módulo `logging` para auditoría completa de todas las peticiones realizadas durante la sesión.

---

## 4. Prerrequisitos

### Conocimiento previo
- Haber completado **Lab 02-00-01** (uso de `requests`, manejo de errores HTTP, variables de entorno).
- Comprensión básica de clases en Python (herencia, métodos, `__init__`).
- Familiaridad con generadores (`yield`) y comprensión de listas.
- Conceptos básicos de bases de datos relacionales (tablas, columnas, claves primarias).

### Acceso y herramientas
- Virtualenv del Lab 02 activo o uno nuevo creado para este laboratorio.
- Conexión a Internet para consultar HackerNews Algolia API y GitHub API.
- Permisos de escritura en el directorio de trabajo del proyecto.
- **No se requieren API keys** para las APIs objetivo de este laboratorio.

---

## 5. Entorno de Laboratorio

### Hardware recomendado

| Componente       | Mínimo                          | Recomendado                     |
|------------------|---------------------------------|---------------------------------|
| RAM              | 4 GB disponibles                | 8 GB                            |
| Almacenamiento   | 500 MB libres                   | 2 GB                            |
| Red              | Conexión a Internet estable     | Conexión a Internet estable     |
| CPU              | 2 núcleos                       | 4 núcleos                       |

### Software requerido

| Paquete           | Versión mínima | Instalación                         |
|-------------------|----------------|--------------------------------------|
| Python            | 3.10+          | Sistema / `pyenv`                    |
| requests          | 2.31+          | `pip install requests`               |
| SQLAlchemy        | 2.0+           | `pip install sqlalchemy`             |
| PyYAML            | 6.0+           | `pip install pyyaml`                 |
| aiohttp           | 3.9+           | `pip install aiohttp`                |
| backoff           | 2.2+           | `pip install backoff`                |

### Configuración inicial del entorno

Ejecuta los siguientes comandos en tu terminal para preparar el entorno antes de comenzar:

```bash
# 1. Crear y activar el virtualenv del laboratorio
python3 -m venv ~/labs/lab03/venv
source ~/labs/lab03/venv/bin/activate   # Linux/macOS
# .\venv\Scripts\activate               # Windows PowerShell

# 2. Instalar dependencias
pip install requests==2.31.0 sqlalchemy==2.0.30 pyyaml==6.0.1 \
            aiohttp==3.9.5 backoff==2.2.1

# 3. Crear estructura de directorios del proyecto
mkdir -p ~/labs/lab03/{config,db,modules,logs,tests}
cd ~/labs/lab03

# 4. Verificar instalaciones
python -c "import requests, sqlalchemy, yaml, aiohttp, backoff; print('OK')"
```

**Salida esperada de verificación:**
```
OK
```

> ⚠️ **Nota ética:** Durante este laboratorio realizarás peticiones reales a APIs públicas. Respeta los delays configurados y no ejecutes los scripts más de una vez por sesión para no saturar los servicios gratuitos.

---

## 6. Desarrollo Paso a Paso

---

### Paso 1 — Crear el módulo de configuración YAML

**Objetivo:** Centralizar todos los parámetros éticos y operacionales en un único archivo de configuración versionable (sin secretos).

#### Instrucciones

**1.1** Crea el archivo de configuración principal:

```bash
cat > ~/labs/lab03/config/settings.yaml << 'EOF'
# ============================================================
# Configuración ética del sistema de consultas - Lab 03-00-01
# ============================================================

http:
  timeout_seconds: 10
  max_retries: 3
  backoff_factor: 2.0          # base para backoff exponencial
  backoff_max_seconds: 30      # techo del delay entre reintentos

rate_limiting:
  delay_between_requests: 1.5  # segundos entre peticiones secuenciales
  delay_between_pages: 1.0     # segundos entre páginas de la misma consulta
  max_concurrent_threads: 3    # máximo de hilos simultáneos
  requests_per_minute: 20      # límite global de peticiones por minuto

pagination:
  default_page_size: 20        # registros por página
  max_records_per_query: 100   # límite de seguridad de registros totales
  max_pages: 10                # límite de páginas por consulta

apis:
  hackernews:
    base_url: "https://hn.algolia.com/api/v1/search"
    tags: "story"
    data_key: "hits"
    mode: "page"
  github:
    base_url: "https://api.github.com/search/repositories"
    data_key: "items"
    mode: "link"
    per_page: 30

logging:
  level: "INFO"
  log_file: "logs/lab03_audit.log"
  format: "%(asctime)s | %(levelname)s | %(name)s | %(message)s"

database:
  url: "sqlite:///db/lab03_results.db"
  echo: false                  # true para ver SQL generado (debug)
EOF
```

**1.2** Crea el módulo Python que carga esta configuración:

```bash
cat > ~/labs/lab03/modules/config_loader.py << 'EOF'
"""
config_loader.py — Carga y valida la configuración YAML del laboratorio.
"""
import yaml
import logging
from pathlib import Path
from typing import Any

_config_cache: dict | None = None


def load_config(path: str = "config/settings.yaml") -> dict:
    """
    Carga el archivo YAML de configuración y lo almacena en caché.
    Lanza FileNotFoundError si el archivo no existe.
    """
    global _config_cache
    if _config_cache is not None:
        return _config_cache

    config_path = Path(path)
    if not config_path.exists():
        raise FileNotFoundError(f"Archivo de configuración no encontrado: {path}")

    with config_path.open("r", encoding="utf-8") as f:
        _config_cache = yaml.safe_load(f)

    return _config_cache


def get(key_path: str, default: Any = None) -> Any:
    """
    Accede a un valor anidado usando notación de punto.
    Ejemplo: get("rate_limiting.delay_between_requests") -> 1.5
    """
    cfg = load_config()
    keys = key_path.split(".")
    value = cfg
    for key in keys:
        if isinstance(value, dict):
            value = value.get(key, default)
        else:
            return default
    return value


def setup_logging() -> logging.Logger:
    """Configura el sistema de logging según settings.yaml."""
    cfg = load_config()
    log_cfg = cfg.get("logging", {})

    log_dir = Path(log_cfg.get("log_file", "logs/app.log")).parent
    log_dir.mkdir(parents=True, exist_ok=True)

    logging.basicConfig(
        level=getattr(logging, log_cfg.get("level", "INFO")),
        format=log_cfg.get("format", "%(asctime)s | %(levelname)s | %(message)s"),
        handlers=[
            logging.FileHandler(log_cfg.get("log_file", "logs/app.log")),
            logging.StreamHandler()
        ]
    )
    return logging.getLogger("lab03")
EOF
```

#### Salida esperada

```bash
cd ~/labs/lab03
python -c "from modules.config_loader import get; print(get('rate_limiting.delay_between_requests'))"
```
```
1.5
```

#### Verificación

```bash
python -c "
from modules.config_loader import load_config, setup_logging
cfg = load_config()
logger = setup_logging()
logger.info('Configuración cargada correctamente')
print('Secciones:', list(cfg.keys()))
"
```

Debes ver en consola y en `logs/lab03_audit.log`:
```
... | INFO | lab03 | Configuración cargada correctamente
Secciones: ['http', 'rate_limiting', 'pagination', 'apis', 'logging', 'database']
```

---

### Paso 2 — Definir el esquema de base de datos con SQLAlchemy ORM

**Objetivo:** Crear un esquema relacional normalizado en SQLite para almacenar resultados de múltiples fuentes.

#### Instrucciones

**2.1** Crea el módulo de modelos ORM:

```bash
cat > ~/labs/lab03/modules/models.py << 'EOF'
"""
models.py — Modelos SQLAlchemy ORM para almacenamiento de resultados.
Esquema normalizado: Source → QuerySession → Record
"""
from datetime import datetime
from sqlalchemy import (
    create_engine, String, Integer, Float, DateTime,
    ForeignKey, Text, Boolean
)
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, mapped_column,
    relationship, Session
)
from modules.config_loader import get


class Base(DeclarativeBase):
    pass


class Source(Base):
    """Fuente de datos (HackerNews, GitHub, etc.)"""
    __tablename__ = "sources"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    base_url: Mapped[str] = mapped_column(String(500), nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime, default=datetime.utcnow
    )
    sessions: Mapped[list["QuerySession"]] = relationship(
        back_populates="source", cascade="all, delete-orphan"
    )


class QuerySession(Base):
    """Sesión de consulta: agrupa todos los registros de una ejecución."""
    __tablename__ = "query_sessions"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    source_id: Mapped[int] = mapped_column(ForeignKey("sources.id"), nullable=False)
    query_term: Mapped[str] = mapped_column(String(500), nullable=False)
    started_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    finished_at: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)
    total_records: Mapped[int] = mapped_column(Integer, default=0)
    pages_fetched: Mapped[int] = mapped_column(Integer, default=0)
    success: Mapped[bool] = mapped_column(Boolean, default=True)

    source: Mapped["Source"] = relationship(back_populates="sessions")
    records: Mapped[list["Record"]] = relationship(
        back_populates="session", cascade="all, delete-orphan"
    )


class Record(Base):
    """Registro individual normalizado."""
    __tablename__ = "records"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    session_id: Mapped[int] = mapped_column(
        ForeignKey("query_sessions.id"), nullable=False
    )
    external_id: Mapped[str | None] = mapped_column(String(200), nullable=True)
    title: Mapped[str | None] = mapped_column(String(1000), nullable=True)
    url: Mapped[str | None] = mapped_column(Text, nullable=True)
    score: Mapped[float | None] = mapped_column(Float, nullable=True)
    author: Mapped[str | None] = mapped_column(String(200), nullable=True)
    source_type: Mapped[str] = mapped_column(String(50), nullable=False)
    raw_data: Mapped[str | None] = mapped_column(Text, nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

    session: Mapped["QuerySession"] = relationship(back_populates="records")


def get_engine():
    """Crea y retorna el engine de SQLAlchemy según la configuración."""
    db_url = get("database.url", "sqlite:///db/lab03_results.db")
    echo = get("database.echo", False)
    return create_engine(db_url, echo=echo)


def init_db():
    """Inicializa la base de datos creando todas las tablas."""
    engine = get_engine()
    Base.metadata.create_all(engine)
    return engine
EOF
```

**2.2** Inicializa la base de datos:

```bash
cd ~/labs/lab03
python -c "
from modules.models import init_db
engine = init_db()
print('Base de datos inicializada en:', engine.url)
"
```

#### Salida esperada

```
Base de datos inicializada en: sqlite:///db/lab03_results.db
```

#### Verificación

```bash
python -c "
import sqlite3
conn = sqlite3.connect('db/lab03_results.db')
cursor = conn.execute(\"SELECT name FROM sqlite_master WHERE type='table'\")
tablas = [row[0] for row in cursor.fetchall()]
print('Tablas creadas:', tablas)
conn.close()
"
```
```
Tablas creadas: ['sources', 'query_sessions', 'records']
```

---

### Paso 3 — Implementar la clase `PaginatedQuery` con reintentos y backoff

**Objetivo:** Construir la clase central de consulta paginada integrando backoff exponencial, logging de auditoría y lectura de parámetros desde la configuración YAML.

#### Instrucciones

**3.1** Crea el módulo de consultas paginadas:

```bash
cat > ~/labs/lab03/modules/paginated_query.py << 'EOF'
"""
paginated_query.py — Clase PaginatedQuery con paginación, reintentos
y backoff exponencial, basada en la Lección 3.1.
"""
import time
import logging
import requests
import backoff
from typing import Generator, Any
from modules.config_loader import load_config, get

logger = logging.getLogger("lab03.paginated_query")


class PaginatedQuery:
    """
    Clase genérica para consultas paginadas a APIs REST.
    Soporta modos: 'page', 'offset', 'cursor', 'link'.
    Integra backoff exponencial, logging de auditoría y
    parámetros éticos desde configuración YAML.
    """

    def __init__(
        self,
        base_url: str,
        params: dict,
        headers: dict = None,
        mode: str = "page",
        page_param: str = "page",
        offset_param: str = "offset",
        limit_param: str = "limit",
        cursor_param: str = "cursor",
        limit: int = None,
        data_key: str = "results",
        cursor_key: str = "next_cursor",
        delay: float = None,
        source_name: str = "unknown"
    ):
        cfg = load_config()
        self.base_url = base_url
        self.params = params.copy()
        self.headers = headers or {}
        self.mode = mode
        self.page_param = page_param
        self.offset_param = offset_param
        self.limit_param = limit_param
        self.cursor_param = cursor_param
        self.limit = limit or cfg["pagination"]["default_page_size"]
        self.data_key = data_key
        self.cursor_key = cursor_key
        # Delay desde config si no se pasa explícitamente
        self.delay = delay if delay is not None else get(
            "rate_limiting.delay_between_pages", 1.0
        )
        self.max_records = get("pagination.max_records_per_query", 100)
        self.max_pages = get("pagination.max_pages", 10)
        self.timeout = get("http.timeout_seconds", 10)
        self.max_retries = get("http.max_retries", 3)
        self.backoff_factor = get("http.backoff_factor", 2.0)
        self.source_name = source_name
        self._session = requests.Session()
        self._session.headers.update(self.headers)
        self._total_fetched = 0
        self._pages_fetched = 0

    # ── Método público principal ──────────────────────────────────────────────
    def fetch_all(self) -> Generator[Any, None, None]:
        """Generador que produce registros individuales de todas las páginas."""
        logger.info(
            "[%s] Iniciando consulta | modo=%s | url=%s | params=%s",
            self.source_name, self.mode, self.base_url,
            {k: v for k, v in self.params.items() if k != "api_key"}
        )
        if self.mode == "page":
            yield from self._paginate_by_page()
        elif self.mode == "offset":
            yield from self._paginate_by_offset()
        elif self.mode == "cursor":
            yield from self._paginate_by_cursor()
        elif self.mode == "link":
            yield from self._paginate_by_link()
        else:
            raise ValueError(f"Modo de paginación desconocido: {self.mode}")
        logger.info(
            "[%s] Consulta finalizada | registros=%d | páginas=%d",
            self.source_name, self._total_fetched, self._pages_fetched
        )

    @property
    def stats(self) -> dict:
        return {
            "total_fetched": self._total_fetched,
            "pages_fetched": self._pages_fetched
        }

    # ── Paginación por número de página ──────────────────────────────────────
    def _paginate_by_page(self):
        page = 1
        while page <= self.max_pages:
            self.params[self.page_param] = page
            registros = self._get_records_with_retry(self.base_url, self.params)
            if not registros:
                logger.debug("[%s] Página %d vacía — deteniendo.", self.source_name, page)
                break
            self._pages_fetched += 1
            for r in registros:
                if self._total_fetched >= self.max_records:
                    logger.info("[%s] Límite de registros alcanzado (%d).",
                                self.source_name, self.max_records)
                    return
                self._total_fetched += 1
                yield r
            page += 1
            time.sleep(self.delay)

    # ── Paginación por offset ─────────────────────────────────────────────────
    def _paginate_by_offset(self):
        offset = 0
        self.params[self.limit_param] = self.limit
        while self._pages_fetched < self.max_pages:
            self.params[self.offset_param] = offset
            registros = self._get_records_with_retry(self.base_url, self.params)
            self._pages_fetched += 1
            for r in registros:
                if self._total_fetched >= self.max_records:
                    return
                self._total_fetched += 1
                yield r
            if len(registros) < self.limit:
                break
            offset += self.limit
            time.sleep(self.delay)

    # ── Paginación por cursor ─────────────────────────────────────────────────
    def _paginate_by_cursor(self):
        cursor = None
        while self._pages_fetched < self.max_pages:
            params = self.params.copy()
            if cursor:
                params[self.cursor_param] = cursor
            response = self._request_with_retry(self.base_url, params)
            body = response.json()
            registros = body.get(self.data_key, [])
            self._pages_fetched += 1
            for r in registros:
                if self._total_fetched >= self.max_records:
                    return
                self._total_fetched += 1
                yield r
            cursor = body.get(self.cursor_key)
            if not cursor:
                break
            time.sleep(self.delay)

    # ── Paginación por cabecera Link ──────────────────────────────────────────
    def _paginate_by_link(self):
        url = self.base_url
        current_params = self.params.copy()
        while url and self._pages_fetched < self.max_pages:
            response = self._request_with_retry(url, current_params)
            body = response.json()
            # GitHub devuelve los items directamente o bajo data_key
            if isinstance(body, list):
                registros = body
            else:
                registros = body.get(self.data_key, [])
            self._pages_fetched += 1
            for r in registros:
                if self._total_fetched >= self.max_records:
                    return
                self._total_fetched += 1
                yield r
            next_link = response.links.get("next", {}).get("url")
            url = next_link
            current_params = {}   # Los parámetros ya van en la URL next
            time.sleep(self.delay)

    # ── Métodos auxiliares con backoff ────────────────────────────────────────
    def _request_with_retry(self, url: str, params: dict) -> requests.Response:
        """Realiza una petición GET con reintentos y backoff exponencial."""
        attempt = 0
        wait = 1.0
        while attempt < self.max_retries:
            try:
                response = self._session.get(url, params=params, timeout=self.timeout)
                logger.debug(
                    "[%s] GET %s | status=%d | intento=%d",
                    self.source_name, url, response.status_code, attempt + 1
                )
                if response.status_code == 429:
                    retry_after = int(response.headers.get("Retry-After", wait))
                    logger.warning(
                        "[%s] Rate limit (429) — esperando %ds", self.source_name, retry_after
                    )
                    time.sleep(retry_after)
                    attempt += 1
                    continue
                response.raise_for_status()
                return response
            except requests.exceptions.RequestException as exc:
                attempt += 1
                if attempt >= self.max_retries:
                    logger.error("[%s] Fallo definitivo tras %d intentos: %s",
                                 self.source_name, self.max_retries, exc)
                    raise
                logger.warning(
                    "[%s] Error en petición (intento %d/%d): %s — backoff %.1fs",
                    self.source_name, attempt, self.max_retries, exc, wait
                )
                time.sleep(min(wait, get("http.backoff_max_seconds", 30)))
                wait *= self.backoff_factor

    def _get_records_with_retry(self, url: str, params: dict) -> list:
        """Wrapper que extrae la lista de registros de la respuesta."""
        response = self._request_with_retry(url, params)
        body = response.json()
        if isinstance(body, list):
            return body
        return body.get(self.data_key, [])

    def __del__(self):
        if hasattr(self, "_session"):
            self._session.close()
EOF
```

#### Verificación rápida

```bash
cd ~/labs/lab03
python -c "
from modules.config_loader import setup_logging
from modules.paginated_query import PaginatedQuery
setup_logging()
# Prueba de instanciación sin ejecutar peticiones
q = PaginatedQuery(
    base_url='https://hn.algolia.com/api/v1/search',
    params={'query': 'test'},
    mode='page',
    data_key='hits',
    source_name='test'
)
print('Instancia creada | delay=', q.delay, '| max_records=', q.max_records)
"
```

```
Instancia creada | delay= 1.0 | max_records= 100
```

---

### Paso 4 — Implementar el módulo de persistencia con SQLAlchemy

**Objetivo:** Crear funciones de repositorio que guarden registros de HackerNews y GitHub en la base de datos con el esquema normalizado.

#### Instrucciones

**4.1** Crea el módulo de repositorio:

```bash
cat > ~/labs/lab03/modules/repository.py << 'EOF'
"""
repository.py — Funciones de persistencia usando SQLAlchemy ORM.
Patrón Repository: desacopla la lógica de negocio del acceso a datos.
"""
import json
import logging
from datetime import datetime
from sqlalchemy.orm import Session
from modules.models import Source, QuerySession, Record, get_engine, init_db

logger = logging.getLogger("lab03.repository")


def get_or_create_source(session: Session, name: str, base_url: str) -> Source:
    """Obtiene una fuente existente o la crea si no existe."""
    source = session.query(Source).filter_by(name=name).first()
    if not source:
        source = Source(name=name, base_url=base_url)
        session.add(source)
        session.flush()
        logger.info("Nueva fuente creada: %s", name)
    return source


def create_query_session(
    session: Session, source: Source, query_term: str
) -> QuerySession:
    """Registra el inicio de una sesión de consulta."""
    qs = QuerySession(
        source_id=source.id,
        query_term=query_term,
        started_at=datetime.utcnow()
    )
    session.add(qs)
    session.flush()
    logger.info("Sesión de consulta iniciada | id=%d | término='%s'", qs.id, query_term)
    return qs


def finalize_query_session(
    session: Session, qs: QuerySession, total_records: int,
    pages_fetched: int, success: bool = True
):
    """Actualiza la sesión con estadísticas finales."""
    qs.finished_at = datetime.utcnow()
    qs.total_records = total_records
    qs.pages_fetched = pages_fetched
    qs.success = success
    session.flush()
    logger.info(
        "Sesión finalizada | id=%d | registros=%d | páginas=%d | éxito=%s",
        qs.id, total_records, pages_fetched, success
    )


def save_hackernews_record(
    session: Session, qs: QuerySession, item: dict
) -> Record:
    """Normaliza y guarda un registro de HackerNews."""
    record = Record(
        session_id=qs.id,
        external_id=str(item.get("objectID", "")),
        title=item.get("title") or item.get("story_title", "")[:1000],
        url=item.get("url") or item.get("story_url", ""),
        score=float(item.get("points") or 0),
        author=item.get("author", ""),
        source_type="hackernews",
        raw_data=json.dumps(item, ensure_ascii=False)[:5000]
    )
    session.add(record)
    return record


def save_github_record(
    session: Session, qs: QuerySession, repo: dict
) -> Record:
    """Normaliza y guarda un registro de GitHub."""
    record = Record(
        session_id=qs.id,
        external_id=str(repo.get("id", "")),
        title=repo.get("full_name", ""),
        url=repo.get("html_url", ""),
        score=float(repo.get("stargazers_count") or 0),
        author=repo.get("owner", {}).get("login", ""),
        source_type="github",
        raw_data=json.dumps({
            "description": repo.get("description"),
            "language": repo.get("language"),
            "forks": repo.get("forks_count"),
            "topics": repo.get("topics", [])
        }, ensure_ascii=False)
    )
    session.add(record)
    return record


def query_results(session: Session, source_type: str = None, limit: int = 20):
    """Consulta registros almacenados, opcionalmente filtrando por fuente."""
    q = session.query(Record)
    if source_type:
        q = q.filter_by(source_type=source_type)
    return q.order_by(Record.score.desc()).limit(limit).all()
EOF
```

#### Verificación

```bash
cd ~/labs/lab03
python -c "
from modules.config_loader import setup_logging
from modules.models import init_db, get_engine
from modules.repository import get_or_create_source
from sqlalchemy.orm import Session

setup_logging()
engine = init_db()
with Session(engine) as session:
    src = get_or_create_source(session, 'test_source', 'https://example.com')
    session.commit()
    print('Fuente creada con id:', src.id)
"
```

```
... | INFO | lab03.repository | Nueva fuente creada: test_source
Fuente creada con id: 1
```

---

### Paso 5 — Implementar concurrencia controlada con ThreadPoolExecutor

**Objetivo:** Crear un orquestador que ejecute consultas a múltiples términos en paralelo con máximo 3 hilos y delays éticos.

#### Instrucciones

**5.1** Crea el módulo de orquestación concurrente:

```bash
cat > ~/labs/lab03/modules/concurrent_collector.py << 'EOF'
"""
concurrent_collector.py — Orquestador de consultas concurrentes.
Usa ThreadPoolExecutor con máximo 3 hilos y delays éticos configurables.
"""
import time
import logging
from concurrent.futures import ThreadPoolExecutor, as_completed
from sqlalchemy.orm import Session
from modules.config_loader import get, load_config
from modules.models import get_engine, init_db
from modules.paginated_query import PaginatedQuery
from modules.repository import (
    get_or_create_source, create_query_session,
    finalize_query_session, save_hackernews_record, save_github_record
)

logger = logging.getLogger("lab03.concurrent_collector")


def collect_hackernews(query_term: str) -> dict:
    """
    Recolecta historias de HackerNews para un término de búsqueda.
    Retorna estadísticas de la operación.
    """
    cfg = load_config()
    engine = init_db()
    stats = {"term": query_term, "records": 0, "pages": 0, "error": None}

    try:
        pq = PaginatedQuery(
            base_url=cfg["apis"]["hackernews"]["base_url"],
            params={
                "query": query_term,
                "tags": cfg["apis"]["hackernews"]["tags"]
            },
            mode=cfg["apis"]["hackernews"]["mode"],
            data_key=cfg["apis"]["hackernews"]["data_key"],
            source_name=f"HN:{query_term}"
        )

        with Session(engine) as session:
            source = get_or_create_source(
                session, "HackerNews", cfg["apis"]["hackernews"]["base_url"]
            )
            qs = create_query_session(session, source, query_term)

            for item in pq.fetch_all():
                save_hackernews_record(session, qs, item)

            finalize_query_session(
                session, qs,
                total_records=pq.stats["total_fetched"],
                pages_fetched=pq.stats["pages_fetched"]
            )
            session.commit()

        stats["records"] = pq.stats["total_fetched"]
        stats["pages"] = pq.stats["pages_fetched"]
        logger.info("[HN] Completado | término='%s' | registros=%d",
                    query_term, stats["records"])

    except Exception as exc:
        stats["error"] = str(exc)
        logger.error("[HN] Error en '%s': %s", query_term, exc)

    return stats


def collect_github(query_term: str) -> dict:
    """
    Recolecta repositorios de GitHub para un término de búsqueda.
    Usa paginación por cabecera Link (modo 'link').
    """
    cfg = load_config()
    engine = init_db()
    stats = {"term": query_term, "records": 0, "pages": 0, "error": None}

    try:
        pq = PaginatedQuery(
            base_url=cfg["apis"]["github"]["base_url"],
            params={
                "q": query_term,
                "sort": "stars",
                "order": "desc",
                "per_page": cfg["apis"]["github"]["per_page"]
            },
            headers={"Accept": "application/vnd.github+json",
                     "X-GitHub-Api-Version": "2022-11-28"},
            mode=cfg["apis"]["github"]["mode"],
            data_key=cfg["apis"]["github"]["data_key"],
            source_name=f"GH:{query_term}"
        )

        with Session(engine) as session:
            source = get_or_create_source(
                session, "GitHub", cfg["apis"]["github"]["base_url"]
            )
            qs = create_query_session(session, source, query_term)

            for repo in pq.fetch_all():
                save_github_record(session, qs, repo)

            finalize_query_session(
                session, qs,
                total_records=pq.stats["total_fetched"],
                pages_fetched=pq.stats["pages_fetched"]
            )
            session.commit()

        stats["records"] = pq.stats["total_fetched"]
        stats["pages"] = pq.stats["pages_fetched"]
        logger.info("[GH] Completado | término='%s' | registros=%d",
                    query_term, stats["records"])

    except Exception as exc:
        stats["error"] = str(exc)
        logger.error("[GH] Error en '%s': %s", query_term, exc)

    return stats


def run_concurrent_collection(
    hn_terms: list[str],
    gh_terms: list[str],
    max_workers: int = None
) -> list[dict]:
    """
    Ejecuta recolección concurrente de HackerNews y GitHub.
    Respeta el límite de hilos configurado en settings.yaml.
    """
    if max_workers is None:
        max_workers = get("rate_limiting.max_concurrent_threads", 3)

    # Construir lista de tareas: (función, argumento)
    tasks = (
        [(collect_hackernews, term) for term in hn_terms] +
        [(collect_github, term) for term in gh_terms]
    )

    delay_between = get("rate_limiting.delay_between_requests", 1.5)
    all_results = []

    logger.info(
        "Iniciando recolección concurrente | hilos=%d | tareas=%d",
        max_workers, len(tasks)
    )

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        future_map = {}
        for i, (fn, term) in enumerate(tasks):
            # Delay escalonado para no saturar en el arranque
            if i > 0:
                time.sleep(delay_between / max_workers)
            future = executor.submit(fn, term)
            future_map[future] = (fn.__name__, term)

        for future in as_completed(future_map):
            fn_name, term = future_map[future]
            try:
                result = future.result()
                all_results.append(result)
                logger.info("Tarea completada | fn=%s | term='%s' | resultado=%s",
                            fn_name, term, result)
            except Exception as exc:
                logger.error("Tarea fallida | fn=%s | term='%s' | error=%s",
                             fn_name, term, exc)
                all_results.append({"term": term, "error": str(exc)})

    return all_results
EOF
```

#### Verificación de sintaxis

```bash
cd ~/labs/lab03
python -c "from modules.concurrent_collector import run_concurrent_collection; print('Módulo cargado OK')"
```
```
Módulo cargado OK
```

---

### Paso 6 — Crear el script principal de ejecución

**Objetivo:** Integrar todos los módulos en un script ejecutable que demuestre el sistema completo funcionando.

#### Instrucciones

**6.1** Crea el script principal:

```bash
cat > ~/labs/lab03/main_collector.py << 'EOF'
#!/usr/bin/env python3
"""
main_collector.py — Script principal del Lab 03-00-01.
Ejecuta recolección concurrente ética de HackerNews y GitHub,
almacena resultados en SQLite y genera un reporte de sesión.
"""
import time
import logging
from modules.config_loader import setup_logging, get
from modules.models import init_db, get_engine
from modules.repository import query_results
from modules.concurrent_collector import run_concurrent_collection
from sqlalchemy.orm import Session


def print_report(engine):
    """Imprime un resumen de los datos almacenados en la base de datos."""
    with Session(engine) as session:
        hn_records = query_results(session, source_type="hackernews", limit=5)
        gh_records = query_results(session, source_type="github", limit=5)

    print("\n" + "=" * 65)
    print("  REPORTE DE RECOLECCIÓN — Lab 03-00-01")
    print("=" * 65)

    print("\n📰 Top 5 HackerNews (por puntos):")
    for r in hn_records:
        title = (r.title or "Sin título")[:55]
        print(f"  [{int(r.score):>5} pts] {title}")

    print("\n⭐ Top 5 GitHub (por estrellas):")
    for r in gh_records:
        title = (r.title or "Sin nombre")[:55]
        print(f"  [{int(r.score):>6} ★] {title}")

    print("\n" + "=" * 65)


def main():
    # 1. Inicializar logging y base de datos
    logger = setup_logging()
    engine = init_db()
    logger.info("Sistema inicializado — Lab 03-00-01")

    # 2. Definir términos de búsqueda
    hn_terms = ["python security", "ethical hacking"]
    gh_terms = ["network scanner python"]

    logger.info("Términos HN: %s | Términos GH: %s", hn_terms, gh_terms)

    # 3. Ejecutar recolección concurrente
    start = time.time()
    results = run_concurrent_collection(
        hn_terms=hn_terms,
        gh_terms=gh_terms
    )
    elapsed = time.time() - start

    # 4. Mostrar estadísticas de ejecución
    print(f"\n⏱  Tiempo total: {elapsed:.1f}s")
    total_records = sum(r.get("records", 0) for r in results)
    errors = [r for r in results if r.get("error")]
    print(f"📦 Registros recolectados: {total_records}")
    print(f"❌ Errores: {len(errors)}")
    if errors:
        for e in errors:
            print(f"   → {e['term']}: {e['error']}")

    # 5. Mostrar reporte desde la base de datos
    print_report(engine)

    logger.info("Ejecución completada | registros=%d | tiempo=%.1fs",
                total_records, elapsed)


if __name__ == "__main__":
    main()
EOF
chmod +x ~/labs/lab03/main_collector.py
```

**6.2** Ejecuta el script principal:

```bash
cd ~/labs/lab03
python main_collector.py
```

#### Salida esperada (ejemplo representativo)

```
2024-01-15 10:23:01 | INFO | lab03 | Sistema inicializado — Lab 03-00-01
2024-01-15 10:23:01 | INFO | lab03.concurrent_collector | Iniciando recolección concurrente | hilos=3 | tareas=3
2024-01-15 10:23:01 | INFO | lab03.repository | Nueva fuente creada: HackerNews
2024-01-15 10:23:01 | INFO | lab03.repository | Nueva fuente creada: GitHub
...
2024-01-15 10:23:28 | INFO | lab03.concurrent_collector | Tarea completada | fn=collect_hackernews | term='python security' | resultado={'term': 'python security', 'records': 20, 'pages': 1, 'error': None}
...

⏱  Tiempo total: 31.4s
📦 Registros recolectados: 60
❌ Errores: 0

=================================================================
  REPORTE DE RECOLECCIÓN — Lab 03-00-01
=================================================================

📰 Top 5 HackerNews (por puntos):
  [  847 pts] Ask HN: Best Python security libraries in 2024
  [  612 pts] Python-based network scanner wins DEF CON award
  ...

⭐ Top 5 GitHub (por estrellas):
  [ 52341 ★] securitytool/awesome-python-security
  [ 31892 ★] networkscanner/nmap-python-wrapper
  ...
=================================================================
```

> ⚠️ Los valores exactos de puntos y estrellas variarán según el estado actual de las APIs. Lo importante es que la estructura del reporte aparezca sin errores.

---

### Paso 7 — Escribir pruebas básicas de validación

**Objetivo:** Verificar que los componentes críticos funcionan correctamente de forma aislada.

#### Instrucciones

**7.1** Crea el archivo de pruebas:

```bash
cat > ~/labs/lab03/tests/test_components.py << 'EOF'
"""
test_components.py — Pruebas de validación del Lab 03-00-01.
Ejecutar con: python -m pytest tests/ -v
"""
import pytest
import os
import sys

# Asegurar que el directorio raíz está en el path
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))


class TestConfigLoader:
    def test_load_config_returns_dict(self):
        from modules.config_loader import load_config
        cfg = load_config()
        assert isinstance(cfg, dict)

    def test_get_nested_value(self):
        from modules.config_loader import get
        delay = get("rate_limiting.delay_between_requests")
        assert isinstance(delay, float)
        assert delay > 0

    def test_get_default_on_missing_key(self):
        from modules.config_loader import get
        result = get("clave.que.no.existe", default="fallback")
        assert result == "fallback"

    def test_max_concurrent_threads_limit(self):
        from modules.config_loader import get
        threads = get("rate_limiting.max_concurrent_threads")
        assert threads <= 3, "No debe exceder 3 hilos para respetar las APIs"


class TestModels:
    def test_init_db_creates_tables(self, tmp_path):
        """Verifica que init_db crea las tres tablas esperadas."""
        import sqlite3
        from sqlalchemy import create_engine
        from modules.models import Base

        db_path = tmp_path / "test.db"
        engine = create_engine(f"sqlite:///{db_path}")
        Base.metadata.create_all(engine)

        conn = sqlite3.connect(str(db_path))
        cursor = conn.execute("SELECT name FROM sqlite_master WHERE type='table'")
        tables = {row[0] for row in cursor.fetchall()}
        conn.close()

        assert "sources" in tables
        assert "query_sessions" in tables
        assert "records" in tables

    def test_record_source_type_field(self, tmp_path):
        """Verifica que un Record se puede crear con source_type."""
        from sqlalchemy import create_engine
        from sqlalchemy.orm import Session
        from modules.models import Base, Record

        engine = create_engine(f"sqlite:///{tmp_path}/test2.db")
        Base.metadata.create_all(engine)

        with Session(engine) as session:
            r = Record(session_id=1, source_type="hackernews", title="Test")
            session.add(r)
            # No hacemos commit para no necesitar FK válida; solo verificamos el objeto
            assert r.source_type == "hackernews"


class TestPaginatedQuery:
    def test_instantiation_with_defaults(self):
        """Verifica que PaginatedQuery se instancia sin errores."""
        from modules.paginated_query import PaginatedQuery
        pq = PaginatedQuery(
            base_url="https://example.com",
            params={"q": "test"},
            mode="page",
            data_key="results"
        )
        assert pq.max_records == 100
        assert pq.delay >= 0

    def test_invalid_mode_raises_error(self):
        """Verifica que un modo inválido lanza ValueError."""
        from modules.paginated_query import PaginatedQuery
        pq = PaginatedQuery(
            base_url="https://example.com",
            params={},
            mode="invalid_mode"
        )
        with pytest.raises(ValueError, match="Modo de paginación desconocido"):
            list(pq.fetch_all())

    def test_stats_initial_values(self):
        """Verifica que las estadísticas iniciales son cero."""
        from modules.paginated_query import PaginatedQuery
        pq = PaginatedQuery(
            base_url="https://example.com",
            params={},
            mode="page"
        )
        assert pq.stats["total_fetched"] == 0
        assert pq.stats["pages_fetched"] == 0


class TestRepository:
    def test_get_or_create_source(self, tmp_path):
        """Verifica idempotencia de get_or_create_source."""
        from sqlalchemy import create_engine
        from sqlalchemy.orm import Session
        from modules.models import Base
        from modules.repository import get_or_create_source

        engine = create_engine(f"sqlite:///{tmp_path}/repo_test.db")
        Base.metadata.create_all(engine)

        with Session(engine) as session:
            s1 = get_or_create_source(session, "TestAPI", "https://test.com")
            session.commit()
            s2 = get_or_create_source(session, "TestAPI", "https://test.com")
            session.commit()
            # Debe retornar el mismo registro, no crear uno nuevo
            assert s1.id == s2.id
EOF
```

**7.2** Ejecuta las pruebas:

```bash
cd ~/labs/lab03
pip install pytest -q
python -m pytest tests/test_components.py -v
```

---

## 7. Validación y Pruebas

### Lista de verificación final

Ejecuta cada comando y confirma que el resultado coincide con lo esperado:

```bash
cd ~/labs/lab03

# 1. Verificar que la base de datos contiene registros
python -c "
import sqlite3
conn = sqlite3.connect('db/lab03_results.db')
for table in ['sources', 'query_sessions', 'records']:
    count = conn.execute(f'SELECT COUNT(*) FROM {table}').fetchone()[0]
    print(f'{table}: {count} filas')
conn.close()
"
```

**Salida esperada:**
```
sources: 2 filas
query_sessions: 3 filas
records: (entre 30 y 100 filas)
```

```bash
# 2. Verificar que el log de auditoría tiene entradas
tail -20 logs/lab03_audit.log | grep -E "(INFO|WARNING|ERROR)"
```

**Salida esperada:** Mínimo 10 líneas con timestamps y nivel `INFO`.

```bash
# 3. Verificar que no hay API keys en el código
grep -r "api_key\s*=\s*['\"][a-zA-Z0-9]" modules/ && echo "ALERTA: API key encontrada" || echo "OK: Sin credenciales hardcodeadas"
```

**Salida esperada:**
```
OK: Sin credenciales hardcodeadas
```

```bash
# 4. Ejecutar suite de pruebas completa
python -m pytest tests/test_components.py -v --tb=short
```

**Salida esperada:**
```
tests/test_components.py::TestConfigLoader::test_load_config_returns_dict PASSED
tests/test_components.py::TestConfigLoader::test_get_nested_value PASSED
tests/test_components.py::TestConfigLoader::test_get_default_on_missing_key PASSED
tests/test_components.py::TestConfigLoader::test_max_concurrent_threads_limit PASSED
tests/test_components.py::TestModels::test_init_db_creates_tables PASSED
tests/test_components.py::TestModels::test_record_source_type_field PASSED
tests/test_components.py::TestPaginatedQuery::test_instantiation_with_defaults PASSED
tests/test_components.py::TestPaginatedQuery::test_invalid_mode_raises_error PASSED
tests/test_components.py::TestPaginatedQuery::test_stats_initial_values PASSED
tests/test_components.py::TestRepository::test_get_or_create_source PASSED

10 passed in X.XXs
```

```bash
# 5. Verificar estructura de archivos del proyecto
find ~/labs/lab03 -name "*.py" -o -name "*.yaml" -o -name "*.db" -o -name "*.log" | sort
```

**Salida esperada (estructura mínima):**
```
~/labs/lab03/config/settings.yaml
~/labs/lab03/db/lab03_results.db
~/labs/lab03/logs/lab03_audit.log
~/labs/lab03/main_collector.py
~/labs/lab03/modules/concurrent_collector.py
~/labs/lab03/modules/config_loader.py
~/labs/lab03/modules/models.py
~/labs/lab03/modules/paginated_query.py
~/labs/lab03/modules/repository.py
~/labs/lab03/tests/test_components.py
```

---

## 8. Solución de Problemas

### Problema 1: `requests.exceptions.ConnectionError` o `HTTPError 403` al consultar GitHub API

**Síntomas:**
```
requests.exceptions.HTTPError: 403 Client Error: rate limit exceeded for url: ...
```
o bien el script se detiene en la primera petición a GitHub con un error de conexión rechazada.

**Causa:**
La GitHub API sin autenticación permite solo **60 peticiones por hora** desde una misma IP. Si ejecutaste el script más de una vez en la misma hora, o si tu IP comparte cuota con otros usuarios (redes universitarias), el límite se agota rápidamente. El código devuelve 403 en lugar de 429 en este caso específico.

**Solución:**

1. Espera que el límite se restablezca (verifica cuándo con el siguiente comando):
   ```bash
   curl -s https://api.github.com/rate_limit | python3 -m json.tool | grep -A5 '"core"'
   ```
2. Si el campo `remaining` es 0, espera hasta el timestamp indicado en `reset`.
3. Para evitar este problema en futuras ejecuciones, añade un token de GitHub (sin scopes especiales) como variable de entorno:
   ```bash
   export GITHUB_TOKEN="ghp_tu_token_aqui"
   ```
   Y modifica `concurrent_collector.py` en la función `collect_github` para leer el token:
   ```python
   import os
   token = os.getenv("GITHUB_TOKEN", "")
   headers = {"Accept": "application/vnd.github+json"}
   if token:
       headers["Authorization"] = f"Bearer {token}"
   ```
4. Reduce `max_records_per_query` a `30` en `settings.yaml` para minimizar el número de peticiones.

---

### Problema 2: `sqlalchemy.exc.OperationalError: no such table` al ejecutar `main_collector.py`

**Síntomas:**
```
sqlalchemy.exc.OperationalError: (sqlite3.OperationalError) no such table: sources
```
El script falla al intentar insertar registros en la base de datos.

**Causa:**
El archivo `db/lab03_results.db` existe pero fue creado antes de que se definieran los modelos correctamente, o el directorio `db/` no existe en el directorio de trabajo actual. SQLAlchemy usa rutas relativas para SQLite, por lo que si el script se ejecuta desde un directorio diferente a `~/labs/lab03`, la base de datos se crea en una ubicación incorrecta.

**Solución:**

1. Asegúrate de ejecutar siempre el script desde el directorio raíz del proyecto:
   ```bash
   cd ~/labs/lab03
   python main_collector.py
   ```
2. Si el problema persiste, elimina la base de datos corrupta y reinicialízala:
   ```bash
   rm -f ~/labs/lab03/db/lab03_results.db
   mkdir -p ~/labs/lab03/db
   python -c "from modules.models import init_db; init_db(); print('DB recreada')"
   ```
3. Para hacer la ruta robusta independientemente del directorio de ejecución, modifica `settings.yaml` para usar una ruta absoluta:
   ```bash
   # Obtener la ruta absoluta
   ABS_PATH=$(realpath ~/labs/lab03)
   # Actualizar settings.yaml
   sed -i "s|sqlite:///db/|sqlite:///${ABS_PATH}/db/|" ~/labs/lab03/config/settings.yaml
   ```
4. Verifica que las tablas existen antes de ejecutar:
   ```bash
   python -c "
   import sqlite3
   conn = sqlite3.connect('db/lab03_results.db')
   tables = conn.execute(\"SELECT name FROM sqlite_master WHERE type='table'\").fetchall()
   print('Tablas:', tables)
   "
   ```

---

## 9. Limpieza del Entorno

Ejecuta los siguientes pasos al finalizar el laboratorio para dejar el entorno ordenado:

```bash
# 1. Desactivar el virtualenv
deactivate

# 2. (Opcional) Comprimir los resultados para entrega
cd ~
tar -czf lab03_resultados_$(date +%Y%m%d).tar.gz labs/lab03/

# 3. Verificar que no hay credenciales en el código antes de archivar
grep -r "api_key\|password\|secret\|token" ~/labs/lab03/modules/ \
     --include="*.py" | grep -v "os.getenv\|# " \
     && echo "⚠️  REVISAR: posibles credenciales encontradas" \
     || echo "✅ Sin credenciales hardcodeadas"

# 4. Limpiar archivos de caché de Python
find ~/labs/lab03 -type d -name "__pycache__" -exec rm -rf {} + 2>/dev/null
find ~/labs/lab03 -name "*.pyc" -delete 2>/dev/null

# 5. Conservar la base de datos y los logs para revisión del instructor
echo "Archivos conservados para evaluación:"
ls -lh ~/labs/lab03/db/lab03_results.db
ls -lh ~/labs/lab03/logs/lab03_audit.log
```

> 📌 **Para el instructor:** Los archivos `db/lab03_results.db` y `logs/lab03_audit.log` son los artefactos de evaluación principales. Verificar que el log contenga entradas de todas las sesiones de consulta y que la base de datos tenga las tres tablas con registros.

---

## 10. Resumen

En este laboratorio construiste un **sistema de recolección de datos ético y robusto** que integra todos los conceptos de la Lección 3.1 en un toolkit funcional:

| Componente               | Módulo                     | Técnica aplicada                              |
|--------------------------|----------------------------|-----------------------------------------------|
| Configuración centralizada | `config_loader.py`        | PyYAML, caché en memoria, acceso por punto    |
| Esquema de datos         | `models.py`                | SQLAlchemy ORM 2.0, relaciones, mapped_column |
| Consultas paginadas      | `paginated_query.py`       | Generadores, 4 modos de paginación, backoff   |
| Persistencia             | `repository.py`            | Patrón Repository, normalización de registros |
| Concurrencia controlada  | `concurrent_collector.py`  | ThreadPoolExecutor, delays éticos, logging    |
| Pruebas                  | `tests/test_components.py` | pytest, fixtures tmp_path, assertions         |

### Principios éticos aplicados

- **Delays configurables** entre peticiones para no saturar los servidores objetivo.
- **Límite máximo** de registros y páginas para evitar bucles infinitos.
- **Máximo 3 hilos** concurrentes, respetando los recursos de las APIs gratuitas.
- **Sin credenciales hardcodeadas**: todas las API keys se leen de variables de entorno.
- **Log de auditoría completo** que registra cada petición realizada.

### Recursos adicionales

- [SQLAlchemy 2.0 — ORM Quickstart](https://docs.sqlalchemy.org/en/20/orm/quickstart.html)
- [Python concurrent.futures — ThreadPoolExecutor](https://docs.python.org/3/library/concurrent.futures.html)
- [HackerNews Algolia API — Documentación oficial](https://hn.algolia.com/api)
- [GitHub REST API — Rate limiting](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)
- [PyYAML — Documentación](https://pyyaml.org/wiki/PyYAMLDocumentation)
- [backoff library — Decoradores de reintento](https://github.com/litl/backoff)

---
