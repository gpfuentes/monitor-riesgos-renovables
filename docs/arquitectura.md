# Arquitectura

## Resumen

Sistema de monitorización de riesgos naturales (incendios, rayos y avisos meteorológicos) sobre los parques renovables de la Península Ibérica.

- **Ingesta:** cada ~15 minutos, GitHub Actions descarga datos de fuentes abiertas y los deposita en Databricks.
- **Procesamiento:** Databricks los transforma siguiendo una arquitectura medallion hasta obtener el riesgo y los MW expuestos por parque.
- **Consumo:** sobre la capa gold se generan alertas (Telegram), un informe en lenguaje natural redactado por un LLM y una vista de negocio en Microsoft Fabric.

---

## Diagrama

```
 ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐
 │  NASA FIRMS  │ │ EUMETSAT MTG │ │    AEMET     │ │  OpenStreetMap   │
 │  REST · CSV  │ │ LI L2·NetCDF │ │  CAP · XML   │ │ Overpass · JSON  │
 └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ └────────┬─────────┘
        └────────────────┴───────┬────────┴──────────────────┘
                                 ▼
             ┌───────────────────────────────────────┐
             │  GitHub Actions  (cron, ~15 min)      │
             │  ingestion/*.py: descarga, filtrado   │
             │  espacial y subida al volumen         │
             └───────────────────┬───────────────────┘
                                 │  Databricks CLI / Files API
 ┌───────────────────────────────▼───────────────────────────────────┐
 │  DATABRICKS · Unity Catalog, catálogo `monitor`                   │
 │                                                                   │
 │  /Volumes/monitor/bronze/landing/<fuente>/…   ficheros originales │
 │        │  job serverless                                          │
 │        ▼                                                          │
 │  bronze.*  ───────────►  silver.*  ────────────►  gold.*          │
 │  Delta, sin transformar   limpieza, deduplicado,   riesgo y MW    │
 │  + metadatos de carga     cruce geoespacial        por parque     │
 └───────────────────────────────┬───────────────────────────────────┘
                                 │  SQL warehouse
        ┌────────────────────────┼────────────────────────┐
        ▼                        ▼                        ▼
 ┌──────────────┐       ┌──────────────────┐     ┌──────────────────┐
 │ Alertas      │       │ Informe          │     │ Microsoft Fabric │
 │ Telegram     │       │ LLM + evals      │     │ Lakehouse +      │
 │ (sin dup.)   │       │ (GitHub Actions) │     │ modelo semántico │
 └──────────────┘       └──────────────────┘     └──────────────────┘
```

---

## Componentes

| Capa | Tecnología | Responsabilidad |
|---|---|---|
| Orquestación e ingesta | GitHub Actions + Python (`ingestion/`) | Planificación, descarga desde las APIs, filtrado previo y subida al volumen de landing. Toda la comunicación con servicios externos. |
| Almacenamiento y gobierno | Databricks Unity Catalog, Delta Lake | Catálogo `monitor` con un esquema por capa; volumen managed para ficheros originales. |
| Transformación | Databricks jobs serverless, PySpark / SQL, funciones H3 (`pipelines/`) | Bronze → silver → gold: tipado, deduplicado, cruce espacial foco–parque y cálculo de riesgo. |
| Alertas | GitHub Actions + bot de Telegram (`alerts/`) | Detección de riesgos nuevos en gold y envío sin duplicados. |
| Informe | LLM vía API + conjunto de evals (`agent/`) | Informe de situación en lenguaje natural a partir de gold, evaluado contra eventos históricos. |
| Negocio | Microsoft Fabric | Lakehouse, modelo semántico y agente de datos sobre las tablas gold. |
| Despliegue | Databricks Asset Bundles + GitHub Actions | Jobs definidos como código; despliegue automático al hacer merge en `main`. |

---

## Flujo de datos

1. **Planificación:** un workflow de GitHub Actions se ejecuta cada ~15 minutos.
2. **Ingesta:** los scripts de `ingestion/` consultan cada fuente. Los datos voluminosos (rayos MTG) se recortan a la Península antes de subirlos.
3. **Landing:** los ficheros se depositan sin modificar en `/Volumes/monitor/bronze/landing/<fuente>/`.
4. **Bronze:** un job de Databricks carga los ficheros nuevos en tablas Delta, sin transformar, añadiendo metadatos de carga (fichero de origen y momento de ingesta).
5. **Silver:**
   - normalización de tipos y deduplicado;
   - filtrado de falsos positivos (detecciones térmicas sobre plantas solares);
   - cruce espacial de cada evento con los polígonos de los parques.
6. **Gold:** agregación por parque, con nivel de riesgo, eventos cercanos y MW expuestos.
7. **Consumo:**
   - GitHub Actions consulta gold a través del SQL warehouse. Si hay riesgos nuevos, envía la alerta y solicita el informe al LLM.
   - Fabric lee gold para el análisis histórico.

---

## Modelo de datos (Unity Catalog)

| Objeto | Tipo | Contenido | Estado |
|---|---|---|---|
| `monitor` | Catálogo | Todo el proyecto | Creado |
| `monitor.bronze` | Esquema | Datos crudos | Creado |
| `monitor.bronze.landing` | Volumen managed | Ficheros originales por fuente | Creado |
| `monitor.bronze.firms_raw` | Tabla Delta | Detecciones FIRMS sin transformar | Prevista (Fase 1) |
| `monitor.silver.parks` | Tabla Delta | Parques solares y eólicos con geometría | Prevista (Fase 1) |
| `monitor.silver.fires` | Tabla Delta | Focos limpios y deduplicados | Prevista (Fase 1) |
| `monitor.silver.fire_park_distances` | Tabla Delta | Pares foco–parque con distancia | Prevista (Fase 1) |
| `monitor.gold.park_risk` | Tabla Delta | Riesgo y MW expuestos por parque | Prevista (Fase 1) |

Los nombres de las tablas previstas son provisionales. Rayos y avisos se incorporan en la Fase 2 con el mismo patrón.

---

## Fuentes de datos

| Fuente | Datos | Acceso | Formato | Particularidades |
|---|---|---|---|---|
| NASA FIRMS | Focos de calor VIIRS / MODIS | API REST con clave gratuita | CSV | Latencia de ~3 h fuera de EE. UU. y Canadá |
| EUMETSAT MTG Lightning Imager | Rayos (L2) | Data Store, cliente `eumdac` | NetCDF | Ficheros voluminosos: se filtran antes de subir |
| AEMET OpenData | Avisos de fenómenos adversos | API con clave gratuita | CAP (XML) | — |
| OpenStreetMap | Parques solares y eólicos | API Overpass (`power=plant`, `plant:source=solar\|wind`) | JSON | Cobertura y etiquetado heterogéneos |

---

## Decisiones de diseño

- **Ingesta desacoplada en GitHub Actions.**
  - La cuota diaria de compute de Databricks no se consume esperando a APIs externas.
  - Los ficheros voluminosos se filtran en origen.
  - Un fallo en una fuente no afecta al procesamiento.
- **Medallion con un esquema por capa.** Permite reprocesar silver y gold desde bronze sin volver a consultar las fuentes, y conceder permisos por capa: los consumidores externos solo leen `gold`.
- **Landing zone como volumen managed.** Conserva los ficheros originales para reprocesado. Es managed porque el entorno (Databricks Free Edition) no permite configurar almacenamiento externo.
- **Salidas desde GitHub Actions.** Alertas y llamadas al LLM se hacen fuera de Databricks, igual que la ingesta: toda la comunicación saliente está en un único punto.
- **`pipelines/` en lugar de `databricks/`.** Así una carpeta local no oculta el paquete `databricks-sdk` al importar.
- **Entorno reproducible.** Python 3.12, alineado con el compute serverless, y dependencias fijadas con `uv` (`pyproject.toml` + `uv.lock`).
- **Gestión de secretos.**
  - El código lee las credenciales de variables de entorno.
  - En CI se guardan en GitHub Secrets; en local, en un `.env` excluido de Git.
  - En desarrollo, la CLI de Databricks se autentica por OAuth.

---

## Estado

| Fase | Alcance | Estado |
|---|---|---|
| 0 | Entorno, Unity Catalog, volumen de landing, prueba de subida | ✅ Prueba completada |
| 1 | Incendios de extremo a extremo (FIRMS + OSM → gold), ejecución programada | Pendiente |
| 2 | Rayos (MTG), avisos (AEMET), cobertura peninsular, filtro de falsos positivos | Pendiente |
| 3 | Alertas en Telegram, informe con LLM y evals | Pendiente |
| 4 | Asset Bundles, CI/CD y tests | Pendiente |
| 5 | Microsoft Fabric: lakehouse, modelo semántico y agente de datos | Pendiente |

---

## Limitaciones conocidas

- **Latencia:** FIRMS publica con ~3 h de retraso en Europa; el sistema no es de detección en tiempo real.
- **Cobertura de parques:** depende de OpenStreetMap. No todos los parques están mapeados y la potencia (MW) no siempre está etiquetada.
- **Falsos positivos:** las plantas solares pueden generar detecciones térmicas; se filtran en silver.
- **Cuota de compute:** Databricks Free Edition limita el consumo diario; los jobs se diseñan ligeros.
- **Integración con Fabric:** la conexión Databricks → Fabric está pendiente de validar en este entorno.
