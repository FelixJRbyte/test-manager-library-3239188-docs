# ESTADO — Manager Library (Laravel + React)

Última actualización: jueves 5 de noviembre de 2026.

**Criterio de honestidad:** cada tarea se marca con una de estas etiquetas.

- `pendiente`: no se ha empezado.
- `implementado, no ejecutado`: el código y los tests están escritos, pero **no se corrieron**.
- `implementado y verificado`: se ejecutó y el resultado se anota aquí.

**Entorno:** esta máquina no tiene PHP, Composer, PostgreSQL ni Docker. Por decisión del instructor no se instalan.
Mientras eso siga así, ninguna tarea del `api` puede quedar como "verificada".

## Fase 0 · Lectura del spec (`spec-.net/`)

| Documento | Estado |
|---|---|
| constitution, spec, plan, architecture, arquitectura-panoramica | leído |
| data-model, api-contract, tasks, adr/001..004 | leído |

## Fase 1 · Traducción (`docs/laravel-react/`)

| Documento | Estado |
|---|---|
| traduccion-laravel-react.md | hecho (documento) |
| decisiones.md (D1..D25) | hecho (documento) |
| tasks.md (T0..T9) | hecho (documento) |
| ADRs adaptados (L001..L005) | hecho (documento) |
| Revisión corta de coherencia (`revision-coherencia.md`, 4 correcciones aplicadas) | hecho (documento) |

## Fase 2 · Implementación

| Tarea | Repo | Estado |
|---|---|---|
| T0 infra + esqueleto + /health | infra, api | **implementado, no ejecutado** (commits en `infra` hasta `feat(infra): T0 servicio api...` y en `api` hasta `05a5d94`) |
| T1 migraciones, errores, login JWT, roles, admin, categorías | api | pendiente |
| T2 usuarios | api | pendiente |
| T3 libros | api | pendiente |
| T4 socios | api | pendiente |
| T5 préstamos | api | pendiente |
| T6 devolución, detalle, listado, vencidos | api | pendiente |
| T7 reporte | api | pendiente |
| T8 frontend | app | pendiente |
| T9 portada, tool, page | api, tool, page | pendiente |

## Deuda, recortes y riesgos abiertos

- **Esqueleto de `api` escrito a mano (sin PHP ni Composer en la máquina):** `composer.json`, `artisan`, `public/index.php`, `bootstrap/app.php`, `bootstrap/providers.php`, `config/database.php`, `config/cors.php` y `phpunit.xml` se escribieron contrastando línea por línea contra las fuentes de `laravel/framework v11.31.0`, `laravel/laravel v11.0.2` y `pest-plugin-arch v3.0.0`, pero **no se ejecutó `composer install` ni `php artisan test`**. No se versiona `composer.lock` por regla de la prueba.
- **Contenedores `infra` y `api` no ejecutados:** no hay Docker instalado en esta máquina; `docker-compose.yml` y `Dockerfile` están revisados contra el contrato y el modelo de datos, pero pendientes de ejecución real.
- **`public.migrations` y `search_path=library,public` (D25):** razonado contra el constructor de esquemas de Laravel, pendiente de correr contra PostgreSQL 16 real.
