# CLAUDE.md · Instrucciones para el agente que construye AcuifIA

AcuifIA es una plataforma SaaS multiempresa de monitoreo de agua subterránea para empresas petroleras, mineras e industriales: ingesta datos de sensores en pozos (nivel, calidad, extracción), los procesa con criterios hidrogeológicos, genera alertas, informes de cumplimiento trazables y recomendaciones con IA, con una web de gestión y una app móvil de campo offline. **Los datos se usan ante autoridades: la integridad, la trazabilidad y la corrección de los cálculos son lo más importante.**

## 1. Documentos (leer antes de programar)

| Orden | Documento | Para qué |
|---|---|---|
| 1 | `docs/README.md` | Índice y decisiones principales |
| 2 | `docs/02-requerimientos-srs.md` | QUÉ construir (RF, HU con criterios, CU, RNF) |
| 3 | `docs/04-arquitectura.md` | Estructura, procesos, contratos, ADRs |
| 4 | `docs/05-diseno-detallado.md` | Fórmulas, QC, base de datos, endpoints, patrones, IA |
| 5 | `docs/anexos/esquema-referencia.sql` | Esquema de base de datos normativo |
| 6 | `docs/03-experiencia-usuario.md` | Pantallas, sistema de diseño, accesibilidad |
| 7 | `docs/06-pruebas.md` | Casos de prueba y oráculos |
| 8 | `docs/07-calidad-configuracion-mantenimiento.md` | Umbrales, DoD, Git, entornos |
| 9 | `docs/08-plan-construccion-fable.md` | Fases y verificaciones |
| 10 | `docs/09-contrato-firmware-sensores.md` + anexos JSON/JS | Contrato con los dispositivos |
| — | `docs/00`, `docs/01` | Contexto de negocio y proceso |

Si dos documentos se contradicen: `02` manda sobre el resto para el **qué**; `05` y el anexo SQL mandan para el **cómo**. Registrá la contradicción en `docs/decisiones-abiertas.md`.

## 2. Reglas de oro

1. **No inventes requerimientos.** Si falta algo, elegí la opción más simple compatible con los documentos y registrala en `docs/decisiones-abiertas.md`.
2. **No cambies arquitectura ni contratos** (API, payload de firmware, esquema de base) sin un ADR propuesto en `docs/adr/`.
3. **Pruebas primero** en `timeseries/pipeline`, `alerts/domain` y `compliance/domain`. Los valores esperados salen de la especificación (fórmulas de `05` §6, casos de `02` HU-24 y de `06`), **nunca** de ejecutar el código.
4. **Nunca** debilites, saltees ni borres una prueba para que pase. Si una prueba está mal, explicalo en el commit y en `docs/progress/ESTADO.md`.
5. **Multi-tenancy siempre:** toda consulta de negocio filtra por `organization_id` además de la RLS; toda ruta nueva con `{id}` entra en la prueba de acceso cruzado.
6. **Datos crudos inmutables:** `ingest_payloads`, `raw_readings`, `audit_log`, `observation_revisions`, `integrity_digests` son de solo inserción.
7. **Las funciones del pipeline son puras:** sin base de datos, sin reloj, sin globals.
8. **La IA solo asesora:** no existe ninguna ruta para controlar bombas (ADR-008). El asistente solo usa herramientas de lectura (ADR-009).
9. **Sin secretos** en código, fixtures, logs ni commits. Servicios externos reales solo si la variable de entorno está configurada; si no, usar los dobles (`ScriptedLLM`, `InMemoryChannel`, `FakeWeatherProvider`).
10. **`make check` en verde** antes de cada commit que cierre una tarea.

## 3. Stack

- **Backend:** Python 3.13, uv, FastAPI, Pydantic v2, SQLAlchemy 2 (async, asyncpg), Alembic, GeoAlchemy2, Celery + Redis, aiomqtt, httpx, structlog, OpenTelemetry, argon2-cffi, PyJWT (Ed25519), pyotp, WeasyPrint + Jinja2, openpyxl, matplotlib (gráficos de informes), pandas/numpy, statsforecast, mlforecast + lightgbm, scikit-learn, ruptures, SDK `anthropic`.
- **Base de datos:** PostgreSQL 17 + TimescaleDB + PostGIS (imagen `timescale/timescaledb-ha`).
- **MQTT:** Eclipse Mosquitto 2 con plugin dynamic-security.
- **Web:** React + TypeScript + Vite, React Router (SPA), TanStack Query/Table, react-hook-form + zod, shadcn/ui + Tailwind, Apache ECharts, MapLibre GL, react-i18next.
- **Móvil:** Expo (React Native) + Expo Router, React Native Paper, expo-sqlite, expo-camera, expo-location, expo-notifications, expo-secure-store.
- **Compartido:** pnpm workspaces; `packages/api-client` generado con orval desde `backend/openapi.json`.
- **Pruebas:** pytest, Hypothesis, testcontainers, schemathesis, pytest-bdd, Vitest, Testing Library, Playwright + axe-core, Jest + RNTL, Maestro, k6.
- **Calidad:** ruff, mypy, import-linter, radon/xenon, jscpd, ESLint, gitleaks, pip-audit, pnpm audit, Trivy, Semgrep.
- Usar las últimas versiones estables compatibles entre sí y **fijarlas** en lockfiles. Antes de usar una API de una librería, verificá que exista en la versión instalada.

## 4. Estructura del repositorio

```
acuifia/
├── CLAUDE.md · README.md · Makefile · .env.example · .pre-commit-config.yaml
├── docs/            especificación, adr/, progress/, runbooks/, sensores/
├── backend/         src/acuifia/<módulo>/{domain,application,infrastructure,api} · alembic/ · tests/
├── simulator/       simulador de dispositivos (CLI acuifia-sim)
├── web/             React
├── mobile/          Expo
├── packages/        api-client · design-tokens · domain-ts
├── infra/           docker-compose*.yml · caddy/ · mosquitto/ · backups/ · observability/
├── firmware/        README que apunta a docs/09 (el firmware lo hace la dueña del proyecto)
└── .github/workflows/
```

Módulos del backend: `shared`, `identity`, `assets`, `devices`, `ingestion`, `timeseries`, `alerts`, `notifications`, `compliance`, `analytics`, `assistant`, `weather`, `audit`, `platform`.

## 5. Comandos

| Comando | Qué hace |
|---|---|
| `make setup` | Instala dependencias (uv, pnpm) y hooks de pre-commit |
| `make up` / `make down` | Levanta / baja el stack local (Docker Compose) |
| `make migrate` | `alembic upgrade head` |
| `make seed` | Semillas (parámetros, perfiles, planes, plantilla genérica) |
| `make demo-reset` | Regenera la organización demo con 1 año de datos simulados |
| `make sim ARGS="--scenario drought --protocol mqtt"` | Corre el simulador contra el stack local |
| `make lint` / `make fmt` | Lint y formato (Python y TS) |
| `make test` | Unitarias + propiedades (todo el monorepo) |
| `make test-int` | Integración y contrato (testcontainers) |
| `make e2e` | Playwright y flujos E2E con simulador |
| `make openapi` | Exporta `backend/openapi.json` y regenera `packages/api-client` |
| `make check` | lint + tipos + import-linter + métricas + test + test-int (lo que corre el CI en un PR) |

## 6. Convenciones

- **Idioma:** código, identificadores, commits y logs en inglés; interfaz, mensajes al usuario y documentación en español (es-AR). Términos de dominio según el glosario de `docs/00` §11.
- **Python:** tipado completo (mypy `--strict` en `domain` y `pipeline`), dataclasses `frozen=True` para valores, sin lógica en routers, inyección de dependencias por constructor, puertos como `typing.Protocol`.
- **Capas:** `api → application → domain ← infrastructure`; `domain` no importa frameworks; ningún módulo importa la `infrastructure` de otro (import-linter lo verifica).
- **Tiempo:** todo en UTC (`timestamptz`); conversión a la zona de la organización solo en la presentación. Nunca `datetime.now()` fuera del puerto `Clock`.
- **Unidades:** se guardan en la unidad canónica del parámetro; la conversión vive en `shared/domain/units.py`.
- **IDs:** UUID v7 generados por la aplicación.
- **Errores de API:** RFC 9457 con `code` estable; mensajes traducibles en el cliente.
- **Logs:** JSON con `request_id`, `trace_id`, `organization_id`; nunca datos personales ni secretos.
- **Frontend:** vistas sin lógica de negocio; ViewModels en hooks; formato de números y fechas con `Intl` es-AR; estados siempre con color + ícono + texto; accesibilidad WCAG 2.2 AA.
- **Base de datos:** una migración por cambio, con `downgrade`; patrón expandir/contraer; nunca editar una migración ya mergeada.
- **Commits:** Conventional Commits (`feat(ingestion): …`). PRs chicos y con referencia a HU/RF.

## 7. Pruebas

- Marcar cada prueba con lo que verifica: `@pytest.mark.req("RF-PRO-03", "HU-24")` y `@pytest.mark.tc("TC-PRO-001")`; en TS, prefijo `[RF-…][TC-…]` en el título.
- Cobertura mínima: ramas ≥ 90 % en dominio (≥ 95 % en `timeseries/pipeline`, `alerts/domain`, `compliance/domain`), líneas ≥ 80 % global.
- Integración con contenedores reales (TimescaleDB, Redis, Mosquitto), no con mocks de base de datos.
- Pruebas de propiedades (Hypothesis) para unidades, hidráulica, volúmenes, agregados e idempotencia (`docs/06` §5.4).

## 8. Definition of Done

Ver `docs/07` §2.4. Resumen: criterios de aceptación con pruebas en verde, `make check` en verde, trazabilidad marcada, OpenAPI y cliente regenerados si cambió la API, migración si cambió el esquema, textos en es-AR y accesibilidad verificada si hay UI, CHANGELOG actualizado.

## 9. Progreso y decisiones

- `docs/progress/ESTADO.md`: fase actual, hecho, pendiente, cómo verificar, cosas para revisión humana. **Actualizar al final de cada fase y antes de que se termine la sesión.**
- `docs/progress/FASE-<n>.md`: plan breve de la fase.
- `docs/decisiones-abiertas.md`: ambigüedades resueltas y supuestos.
- `docs/adr/`: decisiones de arquitectura (nuevas o propuestas de cambio).

## 10. Qué NO hacer

- No agregar microservicios, Kafka, Kubernetes ni otra base de datos (ver ADRs).
- No usar BYPASSRLS fuera del pool de solo lectura del panel de Superadmin.
- No exponer SQL libre ni herramientas de escritura al asistente.
- No enviar datos de una organización a modelos o prompts de otra.
- No borrar datos de cumplimiento; archivar.
- No hardcodear normas, límites legales ni formatos de autoridades: son configuración.
- No agregar dependencias con licencias incompatibles con un SaaS comercial sin ADR.
