# ADR-L003 · PostgreSQL y restricciones del motor, con migraciones de Laravel

- **Estado:** aceptada
- **Adapta:** `spec-.net/adr/adr-001-propiedad-del-esquema.md`, `constitution.md` Arts. V, VI y X, y `data-model.md`.
- **Contexto:** el ADR-001 pone todo el DDL en las migraciones de la aplicación (en .NET, EF Core). El constructor de esquemas de Laravel
  (`Schema::create`) no cubre varias cosas que `data-model.md` exige.
- **Qué cambió y por qué:**

  | Exigencia del data-model | Qué da Laravel | Resolución |
  |---|---|---|
  | Esquema `library` | Usa `public`; crea su tabla `migrations` antes de la primera migración | `DB::statement('CREATE SCHEMA IF NOT EXISTS library')` en la primera migración, `search_path=library,public`, tabla de control `public.migrations` y nombres calificados (D25 en `decisiones.md`) |
  | `CHECK` (`ck_book_available_range`, `ck_book_total_positive`, `ck_user_role`) | No tiene método | `DB::statement('ALTER TABLE ... ADD CONSTRAINT ...')` |
  | Índice único **parcial** de ISBN `WHERE deleted_at IS NULL` | `unique()` no admite `WHERE` | `DB::statement('CREATE UNIQUE INDEX ... WHERE ...')` |
  | Índice parcial `loan_item (due_at) WHERE returned_at IS NULL` | Igual | `DB::statement` |
  | Cero `DEFAULT` | `timestamps()` y otras ayudas pueden poner valores | Se usan solo tipos explícitos; sin `useCurrent()` ni `default()` |
  | Tablas singulares y `user` como palabra reservada | Plural por convención | `$table` explícito; Eloquent entrecomilla `"user"` solo |
  | Semilla de las 5 categorías | Normalmente en seeders | Dentro de la migración inicial, con IDs fijos (data-model §9) |

- **Decisión:** el DDL completo vive en `api/database/migrations/`, en pasos pequeños (esquema y categorías, `book`, `member`, `user`, `loan` y `loan_item`, índices y `CHECK`).
  Se incluyen **desde el principio** `ck_book_total_positive` y `ck_user_role`, que el spec dejaba "pendiente (T-18)" (D9 en `decisiones.md`). `infra/` solo crea el motor vacío.
- **Alternativas descartadas:**
  - *`infra/db/init/*.sql` con el DDL:* segunda fuente de verdad (lo que ADR-001 prohíbe).
  - *Dejar los `CHECK` solo en el dominio PHP:* protege a la aplicación, no a los datos (Art. VI).
  - *Un `structure.sql` volcado:* difícil de revisar y sin historial por cambio.
- **Consecuencias:**
  - Las migraciones dependen de PostgreSQL (SQL crudo): no corren sobre SQLite. Por eso los tests de integración usan PostgreSQL real.
  - Cada `DB::statement` debe tener su `down()` correspondiente.
  - Laravel crea la tabla `migrations`; es del framework, no una entidad de negocio (D14).
  - Una columna con `DEFAULT` en una migración es un defecto.
- **Cómo se verifica:** `Feature/SchemaTest` ejecuta las tres consultas de `data-model.md` §10 y comprueba: las columnas de §3, **ninguna** con `column_default`,
  los tres `CHECK`, el índice parcial de ISBN y las FK con su `ON DELETE`. `grep -ri "create table" infra/` no devuelve nada.
