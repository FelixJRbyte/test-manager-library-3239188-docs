# Revisión corta de coherencia (Fase 1)

**Rol ejercido:** @reviewer (no existen subagentes en `.opencode/agents/`; lo hace el coordinador de forma secuencial, y esto queda declarado).
**Pregunta:** ¿`tasks.md`, la traducción, las decisiones y los ADR contradicen `api-contract.md`, `data-model.md` o la constitución?
**Alcance:** revisión de lectura. No se ejecutó nada.

## Hallazgos y correcciones

| # | Hallazgo | Documento afectado | Gravedad | Corrección |
|---|---|---|---|---|
| R1 | D12 proponía un código `UNAUTHENTICATED` que **no está** en `api-contract.md` §2. El contrato manda sobre el plan y no admite códigos nuevos sin motivo | `decisiones.md` D12, `traduccion` §11, ADR-L001 | Alta | Todo 401 usa `INVALID_CREDENTIALS`; solo cambia el mensaje. Solo se mantiene `INTERNAL_ERROR` (D13), justificado por RNF-4 |
| R2 | `search_path=library` con Laravel: la tabla `migrations` se crea **antes** de la primera migración, cuando `library` aún no existe. El arranque con base vacía (RNF-5) fallaría | D8, ADR-L003, traducción §7 | Alta | D25 nueva: `search_path=library,public`, tabla de control `public.migrations`, nombres calificados |
| R3 | ADR-L005 declaraba una ruta `POST /books/{id}/cover` fuera del contrato; además era innecesaria, porque `_method=PUT` entra por la ruta `PUT` | ADR-L005 | Media | Se elimina la ruta extra. El plan B vive solo en el cliente (React) |
| R4 | T0 pedía un servicio `app` en `docker-compose.yml` con build desde `../app`, pero `app/` no tiene `Dockerfile` hasta T8. El compose quedaría roto en el commit de T0 | `tasks.md` T0 | Media | El servicio `app` se declara con `profiles: ["web"]` y se activa en T8 |

## Verificado sin cambios

| Comprobación | Resultado |
|---|---|
| Los 12 códigos de `api-contract.md` §2 aparecen y se usan con su estado | Coherente |
| Las 6 tablas, columnas y tipos de `data-model.md` §3 se respetan, sin `DEFAULT` ni auditoría | Coherente |
| `ck_book_total_positive` y `ck_user_role` en la migración inicial (D9) | Coherente con Art. VI y X (gana el motor) |
| Estado y `days_late` calculados, sin columna (Art. VII) | Coherente |
| El reporte lee solo `loan_item`, sin join al catálogo (Art. VIII) | Coherente |
| `infra/` sin `CREATE TABLE` (Art. V) | Coherente |
| Ningún secreto versionado: `.env` y `opencode.json` en `.gitignore` (Art. IX) | Coherente |
| Cada HU-01..HU-10 tiene al menos una tarea (mapa de `tasks.md`) | Coherente |
| Cada RNF tiene dónde se comprueba (RNF-1 en T9, RNF-2 en T5, RNF-3 en T1/T2, RNF-4 en T1, RNF-5 en T0) | Coherente |

## Riesgos que quedan abiertos (no se resuelven leyendo)

1. **Concurrencia (ADR-L002):** el spec dice optimista; aquí es pesimista por fila. Se justificó por data-model y se registra en D15. Falta ejecutarlo contra PostgreSQL real.
2. **`PUT` multipart (ADR-L005):** no se sabe si PHP lo parsea en esta versión. El plan B existe, pero no está probado.
3. **D25 (`public.migrations`):** razonado, no ejecutado.
4. **Sin PHP en la máquina:** ningún test del `api` se puede correr. `ESTADO.md` marca el máximo alcanzable como "implementado, no ejecutado".

## Veredicto

Con las cuatro correcciones, no se encuentran contradicciones abiertas con el contrato, el modelo de datos ni la constitución. Se puede pasar a `infra`.
