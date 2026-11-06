# Decisiones — huecos del spec y cómo se resolvieron

Este registro recoge cada punto donde el spec calla, se contradice o deja una decisión técnica abierta.
Regla de resolución: `constitution` > `data-model` (datos) / `api-contract` (HTTP) > `spec` > `plan`. En datos, gana el motor (Art. X).

Cada decisión indica **qué documentos entran en tensión** y **por qué se resolvió así**. Nada se resuelve en silencio.

## Índice

| # | Tema | Tipo |
|---|---|---|
| D1 | Orden de validaciones al prestar | Hueco |
| D2 | `size` mayor que 50 | Hueco |
| D3 | "La base arranca vacía" (RNF-5) | Aclaración |
| D4 | JWT en Laravel | Traducción |
| D5 | Qué reemplaza `PUT /books/{id}` | Hueco |
| D6 | Rango del reporte | Hueco |
| D7 | Límite de 3 libros | Aclaración |
| D8 | Convenciones del data-model en Laravel | Traducción |
| D9 | Restricciones "pendiente (T-18)" | **Contradicción** |
| D10 | `days_late` | Hueco |
| D11 | Formato de fechas | Hueco |
| D12 | Código para 401 sin token | **Contradicción** |
| D13 | Errores no previstos (500) | Hueco |
| D14 | Tablas por defecto de Laravel | **Contradicción** |
| D15 | Concurrencia: optimista vs. bloqueo | **Contradicción** |
| D16 | Validación de entrada: casos sin código en §2 | Hueco |
| D17 | `active_loans` del socio | Hueco |
| D18 | Filtros de `GET /loans` | Hueco |
| D19 | Ruta de `/health` | Hueco |
| D20 | Portada en el navegador | Hueco |
| D21 | `POST /users`: campo `role` | Hueco |
| D22 | Conteo de "sin devolver" y vencidos | Aclaración |
| D23 | Paginación fuera de rango | Hueco |
| D24 | Identificador que no es UUID | Hueco |
| D25 | Tabla `migrations` y `search_path` | Riesgo técnico |

---

## D1 · Orden de validaciones al prestar

**Hueco.** `spec.md` CA-06.1 a CA-06.5 enumera las reglas, pero no dice cuál se evalúa primero cuando se rompen varias a la vez.
El resultado visible cambia (409 o 422, y qué `code`).

**Decisión.** Se evalúa en este orden y se detiene en el primer fallo:

1. **422 `VALIDATION_ERROR`** de forma: `member_id` UUID, `items` de 1 a 3 elementos, cada `book_id` UUID.
2. **422 `DUPLICATE_BOOK_IN_LOAN`** si un `book_id` se repite (CA-06.2).
3. **404 `NOT_FOUND`** si el socio o algún libro no existe o está dado de baja.
4. **409 `MEMBER_HAS_OVERDUE`** si el socio tiene alguna línea vencida sin devolver (CA-06.5).
5. **422 `MEMBER_LIMIT_EXCEEDED`** si `sin devolver + líneas nuevas > 3`. El mensaje incluye el conteo actual (CA-06.4).
6. **409 `BOOK_NOT_AVAILABLE`** si algún libro no tiene ejemplares. Se evalúa **dentro de la transacción** con el libro bloqueado. Si falla, no se registra nada (CA-06.3).

**Por qué.** Las reglas baratas y que no dependen del estado concurrente van primero. La de ejemplares va al final y dentro de la transacción,
porque es la única que otra petición puede cambiar entre la lectura y la escritura.
El vencido va antes que el límite porque el spec lo trata como bloqueo total (RN-3: "no pide más hasta devolverlos").

## D2 · `size` mayor que 50

**Hueco.** `api-contract` §1 dice "`size` máximo 50" sin decir qué pasa al pasarse.

**Decisión.** `size > 50` responde **422 `VALIDATION_ERROR`** con `details: [{field: "size", issue: "máximo 50"}]`. No se recorta en silencio.

**Por qué.** Recortar oculta al cliente que su petición no se cumplió como la pidió. Falla ruidosamente (mismo criterio del Art. VI).

## D3 · "La base arranca vacía" (RNF-5)

**Aclaración.** `spec.md` RNF-5 pide "la base arranca vacía". `data-model.md` §9 exige las 5 categorías en la migración inicial.
Son compatibles si "vacía" significa **sin datos de negocio**.

**Decisión.**
- El motor arranca **sin tablas** (lo hace `infra`).
- La migración inicial crea el esquema y siembra las **5 categorías con IDs fijos** (data-model §9).
- El admin lo crea el comando idempotente `php artisan library:ensure-admin`, que lee `ADMIN_USERNAME` y `ADMIN_PASSWORD` del entorno y usa el puerto `PasswordHasher`. Si el usuario ya existe, no hace nada.
- Libros y socios de demostración los pone `tool/` **usando la API**, nunca SQL (data-model §9).

**Por qué.** La base no puede producir el hash de la clave (plan D-10). La categoría es un dato de referencia, no de negocio.

## D4 · JWT en Laravel

**Traducción.** `api-contract` §4 exige "JWT firmado, 8 horas, con `sub` y `role`". Laravel Sanctum emite tokens opacos, no JWT.

**Decisión.** `firebase/php-jwt` (HS256) detrás del puerto `TokenIssuer`, con un middleware propio. Secreto en `JWT_SECRET`.
Claims: `sub` (id del usuario), `role`, `iat`, `exp = iat + 28800`. Sin datos personales. Ver [ADR-L001](adr/adr-l001-jwt-propio-vs-sanctum.md).

## D5 · Qué reemplaza `PUT /books/{id}`

**Hueco.** El contrato dice "Reemplaza título, autor, categoría y total". No dice nada del ISBN.

**Decisión.** `PUT` reemplaza `title`, `author`, `category_id` y `total_copies`. **El ISBN no se modifica**: si viene en el cuerpo, se ignora.
Cambiar `total_copies` ajusta `available_copies` en la misma diferencia (CA-03.3). Si quedara `< 0`: 422 `VALIDATION_ERROR` en el campo `total_copies`.

**Por qué.** El ISBN identifica la edición y es la clave de unicidad (CA-03.2). Cambiarlo abriría un `ISBN_TAKEN` que el contrato no prevé en `PUT`.

## D6 · Rango del reporte

**Hueco.** "Fecha, ambos inclusive" no dice cómo comparar una fecha (`2026-10-31`) con un instante (`loaned_at`).

**Decisión.** Incluye `loaned_at >= from 00:00:00Z` y `loaned_at < (to + 1 día) 00:00:00Z`.
- `from` y `to` son obligatorios. Si faltan o no son `YYYY-MM-DD` válidos: 422 `VALIDATION_ERROR`.
- `to < from`: 422 `INVALID_DATE_RANGE` (CA-10.2).
- Rango sin préstamos: 200 con `rows: []`.
- El reporte se cuenta por **línea** (`loan_item`): una línea prestada es una vez prestado.

**Por qué.** Cierra el último día completo sin perder préstamos de las 23:59.

## D7 · Límite de 3 libros

**Aclaración.** CA-06.4 y RN-2: "máximo 3 libros sin devolver por socio".

**Decisión.** Se cuenta por **línea sin devolver** (`returned_at IS NULL`) de todos los préstamos del socio. Si `sin devolver + líneas nuevas > 3`: 422 `MEMBER_LIMIT_EXCEEDED`.
Ejemplo: tiene 2 sin devolver y pide 2 libros → 4 > 3 → 422, con "ya tiene 2" en el mensaje.

## D8 · Convenciones del data-model en Laravel

**Traducción.** `data-model.md` §0 y §3 fijan nombres, tipos y la ausencia de `DEFAULT`. Laravel tiene convenciones distintas.

| Convención de Laravel | Decisión |
|---|---|
| Tablas en plural | Singular, con `protected $table = 'book'` explícito en cada modelo |
| Esquema `public` | Esquema `library`: `CREATE SCHEMA IF NOT EXISTS library` en la primera migración, `search_path=library,public` en `config/database.php` y nombres calificados (ver D25 por qué no basta `search_path=library`) |
| `created_at`/`updated_at` | `public $timestamps = false`; las migraciones no llaman a `timestamps()` |
| Columnas con `DEFAULT` | Ninguna. Los valores los pone el código |
| IDs autoincrementales | UUID **v4** con `Str::uuid()`; `$incrementing = false`; `$keyType = 'string'` |
| `HasUuids` | En Laravel reciente genera **UUID v7**, y el contrato exige v4. Si se usa, se sobrescribe `newUniqueId()` para devolver `(string) Str::uuid()` |
| `softDeletes()` | `book.deleted_at` es `timestamptz` nulo, igual que lo hace el trait |

## D9 · Restricciones "pendiente (T-18)"

**Contradicción.** `data-model.md` §4 marca `ck_book_total_positive` y `ck_user_role` como "pendiente (T-18) · hoy solo dominio".
Pero `constitution.md` Art. VI dice que toda invariante expresable en el motor se declara ahí, y Art. X dice que gana el motor.

**Decisión.** Ambas restricciones van **ya en la migración inicial**. T-18 del spec deja de ser una tarea aparte.
Al implementarlas, `data-model.md` §4 de la versión Laravel las reclasifica como **motor**.

**Por qué.** No tiene sentido promoverlas después si se pueden imponer desde el principio. Aplica la jerarquía: constitución > data-model.

## D10 · `days_late`

**Hueco.** CA-09.1 pide "días de retraso" sin definir el redondeo.

**Decisión.** `days_late` = días **completos** transcurridos desde `due_at` hasta ahora, con **mínimo 1**.
Fórmula: `max(1, intdiv(now - due_at en segundos, 86400))`.

**Por qué.** Solo se listan líneas con `due_at < ahora`; una línea vencida hace 3 horas tiene 0 días completos, pero un vencido con 0 de retraso sería confuso. El mínimo 1 evita ese 0.

## D11 · Formato de fechas

**Hueco.** El contrato muestra `2026-10-14T14:15:00Z` pero Laravel serializa fechas con milisegundos (`.000000Z`).

**Decisión.** `APP_TIMEZONE=UTC`. Toda fecha de salida usa el formato `Y-m-d\TH:i:s\Z`, sin fracciones, forzado en los API Resources.

## D12 · Código para 401 sin token

**Contradicción (aparente).** `api-contract` §1 dice que 401 aplica a "sin token, token inválido o vencido, credenciales inválidas". Pero §2 solo define `INVALID_CREDENTIALS` (401), con origen CA-01.2 (login).
Un token ausente no es, en rigor, una credencial inválida.

**Decisión.** **No se inventa un código nuevo.** Todo 401 usa `INVALID_CREDENTIALS`, que es el único código 401 del §2.
Lo que cambia es el `message` en español: en el login, "Usuario o contraseña incorrectos" (idéntico para usuario inexistente y clave errónea, CA-01.2);
con token ausente, inválido o vencido, "Sesión no válida o vencida. Inicie sesión de nuevo".

**Por qué.** Se aplica la jerarquía: el contrato (§2) lista los códigos válidos y manda sobre cualquier preferencia. El cliente React no necesita distinguirlos:
ante un 401 fuera de `/auth/login` limpia la sesión y vuelve al login.
Una primera versión de este documento proponía `UNAUTHENTICATED`; se descartó en la revisión de coherencia ([`revision-coherencia.md`](revision-coherencia.md)).

## D13 · Errores no previstos

**Hueco.** RNF-4 dice "todo error HTTP usa el mismo cuerpo", y el contrato prohíbe trazas, SQL y rutas. El contrato no define ningún código para un error del servidor.

**Decisión.** Cualquier `Throwable` no mapeada responde **500** con `code: "INTERNAL_ERROR"`, mensaje genérico en español y `details: []`.
Es la **única** extensión de códigos respecto al §2: sin ella, un fallo inesperado rompería RNF-4 (cuerpo único) o filtraría la traza.
El detalle real se registra en el log del servidor, **sin** el cuerpo del login ni contraseñas.
Con `APP_DEBUG=false` en producción.

## D14 · Tablas por defecto de Laravel

**Contradicción.** `data-model.md` §2 dice "Seis entidades, seis tablas". El esqueleto de Laravel trae migraciones de `users`, `password_reset_tokens`, `sessions`, `cache` y `jobs`.

**Decisión.** Esas migraciones **se borran**. Se configuran `CACHE_STORE=array`, `SESSION_DRIVER=array`, `QUEUE_CONNECTION=sync`.
La tabla `migrations` la crea Laravel y es infraestructura del framework, no una entidad de negocio.

## D15 · Concurrencia: optimista vs. bloqueo

**Contradicción.** `plan.md` D-04 y ADR-002 eligen concurrencia **optimista** con testigo de versión y descartan `SELECT … FOR UPDATE`.
Pero `data-model.md` §3 **no tiene** ninguna columna de versión ni de testigo, y la constitución exige que ese documento sea la única fuente del modelo.

**Decisión.** Se usa bloqueo pesimista acotado: `DB::transaction` + `lockForUpdate` sobre los libros del préstamo, **ordenados por id** para evitar interbloqueos.
La última barrera sigue siendo `ck_book_available_range`. Ver [ADR-L002](adr/adr-l002-concurrencia-lockforupdate.md).

**Por qué.** Respeta data-model (sin columna nueva) y cumple RNF-2. El argumento del ADR original ("serializa toda la biblioteca") no aplica: el bloqueo es por fila de libro y solo dura la transacción.

## D16 · Validación de entrada sin código en §2

**Hueco.** El contrato solo tiene `VALIDATION_ERROR` genérico para "cualquier campo inválido".

**Decisión.** Todos estos casos responden **422 `VALIDATION_ERROR`** con `details` por campo:
- `category_id` inexistente al crear o editar un libro (se verifica antes de insertar; la FK sería un 500).
- ISBN con formato inválido.
- `total_copies < 1`.
- Cambio de `total_copies` que dejaría `available_copies < 0`.
- Imagen de portada con tipo o tamaño inválido.
- Campos obligatorios ausentes o vacíos.

Los duplicados que el motor detecta (`ISBN_TAKEN`, `DOCUMENT_TAKEN`, `USERNAME_TAKEN`) se comprueban antes de insertar **y** se captura la violación de unicidad
por si dos peticiones compiten, para devolver el mismo 409 y no un 500.

## D17 · `active_loans` del socio

**Hueco.** `GET /members/{id}` incluye "`active_loans` (conteo sin devolver)". No dice si cuenta préstamos o líneas.

**Decisión.** Es el número de **líneas sin devolver** del socio. Es el mismo número que usa la regla de límite (D7).

## D18 · Filtros de `GET /loans`

**Hueco.** El contrato lista `from`, `to`, `status`, `member_id`, `page`, `size`, sin decir cuáles son obligatorios ni cómo se aplica el rango.

**Decisión.**
- Todos son opcionales.
- `from`/`to` usan la misma semántica inclusiva de D6, sobre `loan.loaned_at`. Si `to < from`: 422 `INVALID_DATE_RANGE`.
- `status` admite `open`, `overdue`, `closed`; otro valor: 422 `VALIDATION_ERROR`.
- Orden: `loaned_at` descendente (CA-08.2).
- El filtro por estado se hace con `whereExists` sobre las líneas, nunca con una columna (Art. VII).

## D19 · Ruta de `/health`

**Hueco.** El contrato lista `GET /health` entre los endpoints, sin prefijo explícito.

**Decisión.** Se sirve en `/api/v1/health` (la base `/api/v1` aplica a todo). Responde `{"status":"ok"}` sin tocar la base.
El `healthcheck` de Docker usa esa ruta.

## D20 · Portada en el navegador

**Hueco.** `GET /books/{id}/cover` exige token (CA-01.3), y una etiqueta `<img src>` no puede enviar el encabezado `Authorization`.

**Decisión.** React pide la imagen con `fetch` y el token, la convierte en `blob:` URL y la pinta. No se hace pública la ruta ni se pone el token en la URL.

## D21 · `POST /users`: campo `role`

**Hueco.** El ejemplo lleva `role: "librarian"`; no dice si es obligatorio.

**Decisión.** `role` es obligatorio. Solo se acepta `librarian`. `admin` responde 422 `VALIDATION_ERROR` (CA-02.3); cualquier otro valor, también.
Si falta, 422.

## D22 · "Sin devolver" y "vencido"

**Aclaración.** Una línea está **sin devolver** si `returned_at IS NULL`. Está **vencida** si además `due_at < ahora` (estrictamente menor).
Un préstamo es `overdue` si alguna de sus líneas está vencida; `open` si tiene líneas sin devolver y ninguna vencida; `closed` si todas están devueltas.
"Ahora" lo da siempre el puerto `Clock`, nunca `now()` suelto dentro del dominio.

## D23 · Paginación fuera de rango

**Hueco.** El contrato no dice qué pasa con `page=0` o `size=0`.

**Decisión.** `page < 1` o `size < 1` → 422 `VALIDATION_ERROR`. Valores por defecto: `page=1`, `size=20`. Una página mayor que la última devuelve `items: []` con el `total` real.

## D24 · Identificador que no es UUID

**Hueco.** `GET /books/abc`: ¿422 o 404?

**Decisión.** **404 `NOT_FOUND`**. Un identificador mal formado no puede corresponder a ningún recurso, y el contrato dice que 404 aplica a "cualquier recurso".
Evita además un error de Postgres por tipo `uuid` inválido (que sería un 500).

## D25 · Tabla `migrations` y `search_path`

**Riesgo técnico (no verificado: no hay PHP en la máquina).** D8 fija `search_path=library`, y la primera migración es la que crea el esquema `library`.
Pero Laravel crea su tabla `migrations` **antes** de correr ninguna migración, y la crea en el primer esquema del `search_path`. Con `search_path=library`
y el esquema aún inexistente, ese paso fallaría y la base nunca arrancaría (rompe RNF-5).

**Decisión.**
- `search_path = library,public` en `config/database.php`.
- La tabla de control se fija a `public.migrations` (`'migrations' => ['table' => 'public.migrations']`), para que su ubicación no dependa de qué esquema exista en cada momento.
- Las migraciones y los modelos usan nombres **calificados** (`library.book`) en lugar de depender del `search_path`.
- `public.migrations` sigue siendo de la infraestructura del framework (D14), no una entidad de negocio.

**Por qué.** Hace el arranque determinista sin poner DDL en `infra/` (Art. V). Queda anotado como **no ejecutado**: se comprueba al primer `migrate` real.
