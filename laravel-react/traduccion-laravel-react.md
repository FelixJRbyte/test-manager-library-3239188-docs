# Traducción del spec .NET a Laravel + React

Este documento recorre `spec-.net/` y separa dos tipos de decisión:

- **Negocio** (historias HU, criterios CA, reglas RN, requisitos RNF, contrato de API y modelo de datos). Se respeta íntegro.
- **Tecnología** (lenguaje, framework, ORM, herramientas de prueba). Se traduce a Laravel y React y se documenta aquí.

Jerarquía cuando dos documentos se contradicen: `constitution` > `data-model` (datos) / `api-contract` (HTTP) > `spec` > `plan`.
En datos, gana el motor (Art. X). Las contradicciones halladas están en [`decisiones.md`](decisiones.md).

## 1. Tabla maestra: decisión del spec → equivalente

| Origen en el spec | Decisión del spec (.NET) | Equivalente en este reto | Tipo |
|---|---|---|---|
| plan D-01 | .NET 8 / C# 12 | PHP 8.3 + Laravel 11 | Tecnología |
| plan D-02 | ASP.NET Core Minimal API | Rutas de Laravel en `routes/api.php`, prefijo `/api/v1` | Tecnología |
| plan D-03, ADR-001 | EF Core + migraciones de EF | Migraciones de Laravel; todo el DDL ahí | Tecnología |
| plan D-04, ADR-002 | Concurrencia optimista con testigo de versión | `DB::transaction` + `lockForUpdate` ordenado por id, con el `CHECK` como última barrera ([ADR-L002](adr/adr-l002-concurrencia-lockforupdate.md)) | Tecnología |
| plan D-05 | `Clock` como puerto, `timestamptz` UTC | Puerto `Clock` + `APP_TIMEZONE=UTC` + columnas `timestampTz` | Tecnología |
| plan D-06, ADR-004 | Reporte con una consulta agregada por puerto de lectura | Puerto `ReportReader` con una consulta agregada sobre `loan_item` | Negocio + tecnología |
| plan D-07 | Objetos de valor `Isbn`, `Quantity`, `DocumentNumber`, `DateRange` | Clases PHP `final readonly` en `Domain\ValueObjects`, solo los que aportan (ver §3) | Tecnología |
| plan D-08 | Portadas en disco/S3, `cover_key` opaca | Puerto `CoverStorage` con adaptador de disco local (`storage/app/covers`) | Tecnología |
| plan D-09 | BCrypt factor 12 detrás de `PasswordHasher` | Adaptador sobre `Hash::make` / `Hash::check` con `BCRYPT_ROUNDS=12` | Tecnología |
| plan D-10 | El arranque crea el admin desde el entorno | Comando artisan idempotente `library:ensure-admin` (D3) | Tecnología |
| plan D-11 | JWT HS256, 8 h, `JWT_SECRET` | `firebase/php-jwt` + middleware propio ([ADR-L001](adr/adr-l001-jwt-propio-vs-sanctum.md)) | Tecnología |
| plan D-12 | xUnit, fakes, base real en integración | Pest, fakes de puertos y PostgreSQL real en integración | Tecnología |
| constitution I | NetArchTest: dominio sin dependencias | `arch()` de Pest: `Domain` no usa `Illuminate`, `Eloquent` ni terceros | Tecnología |
| constitution II | NetArchTest: `application` no importa `adapters` | `arch()`: `Application` no usa `Infrastructure` ni `Http` | Tecnología |
| constitution III | Un único `PortBindings.cs` | Un único `PortBindingsServiceProvider` | Tecnología |
| constitution V | `infra` sin `CREATE TABLE` | Igual; se comprueba con `grep` sobre `infra/` | Negocio |
| constitution VI | Invariantes al motor | `CHECK`, `UNIQUE`, FK en migraciones (D9 las trae todas) | Negocio |
| constitution VII | Estado y `days_late` calculados | Calculados en `Domain\Loan` y en consultas; sin columna | Negocio |
| constitution VIII | Valores congelados | Columnas `book_title`, `isbn`, `category_name` en `loan_item` | Negocio |
| constitution IX | Sin secretos | `.env` ignorado, solo `.env.example`; `password` y `password_hash` ocultos | Negocio |
| constitution XI | Idioma | Código en inglés; mensajes y documentación en español | Negocio |
| architecture §5 | Errores propios traducidos solo por el adaptador HTTP | `DomainException` propia con `code` y `status` + handler global en `bootstrap/app.php` | Tecnología |
| (validación) | FluentValidation | Form Requests con mensajes en español | Tecnología |
| spec HU-* / CA-* / RN-* / RNF-* | — | Sin cambios | Negocio |
| api-contract completo | — | Sin cambios | Negocio |
| data-model completo | — | Sin cambios (convenciones de Laravel en D8) | Negocio |
| Frontend (tasks T-22) | sin tecnología fijada | React + Vite, `fetch`, token en `sessionStorage`, rutas protegidas por rol | Tecnología |

## 2. Capas y carpetas

```text
api/app/
├── Domain/                  # PHP puro. Sin Illuminate, sin Eloquent, sin librerías
│   ├── Book.php  Loan.php  LoanItem.php  Member.php  User.php
│   ├── ValueObjects/        # Isbn, DateRange
│   └── Exceptions/          # DomainException propia (code, status) y errores de negocio
├── Application/
│   ├── Ports/               # Interfaces que la aplicación define (ver §4)
│   └── Services/            # Un servicio por caso de uso
├── Infrastructure/
│   ├── Persistence/         # Modelos Eloquent y repositorios que implementan los puertos
│   ├── Security/            # BcryptPasswordHasher, JwtTokenIssuer
│   ├── Storage/             # LocalCoverStorage
│   └── Support/             # SystemClock, DbTransactionRunner
├── Http/
│   ├── Controllers/         # Delgados: validan, llaman al servicio, devuelven Resource
│   ├── Requests/            # Form Requests con mensajes en español
│   ├── Resources/           # API Resources; forzan formato de fecha (D11)
│   └── Middleware/          # JwtAuthenticate, RequireRole
└── Providers/
    └── PortBindingsServiceProvider.php   # ÚNICO punto de composición
```

Dependencias permitidas (flechas hacia adentro):

```text
Http ──► Application ──► Domain
Infrastructure ──► Application (implementa sus puertos) ──► Domain
Providers ──► todas (único que ve ambas mitades)
```

### Compromiso de ligereza (4 horas)

El spec .NET separa muchas clases. Aquí se mantiene la **frontera**, no la cantidad:

- Un servicio por caso de uso, pero solo los que el contrato exige (ver §5).
- Los objetos de valor se reducen a `Isbn` y `DateRange`. `Quantity` y `DocumentNumber` se validan dentro de `Book` y `Member`.
- Los modelos Eloquent viven en `Infrastructure\Persistence`. El dominio no los conoce: los repositorios convierten entre modelo y entidad.

## 3. Dominio: qué hace cada clase pura

| Clase | Invariantes que defiende | Origen |
|---|---|---|
| `Book` | `0 <= available <= total`; `total >= 1`; título no vacío y recortado; `withdraw()` y `restock()`; cambio de total ajusta disponibles en la misma diferencia (422 si queda `< 0`) | CA-03.3, RN-1 |
| `Loan` | 1 a 3 líneas; sin libro repetido; `due_at = loaned_at + 14 días`; `addItem` llama a `Book::withdraw`; `returnItem` llama a `Book::restock` una sola vez; estado calculado | CA-06.*, CA-07.*, RN-2 |
| `LoanItem` | Congela título, ISBN y categoría; `returned_at` nulo hasta la devolución | RN-4 |
| `Member` | Nombre y documento obligatorios | CA-05.1 |
| `User` | Nombre en minúsculas y recortado; rol en (`admin`, `librarian`) | CA-02.2 |
| `Isbn` | Exactamente 13 dígitos | CA-03.2 |
| `DateRange` | `to >= from` | CA-10.2 |

## 4. Puertos (ISP: pequeños, sin tipos de Laravel)

Los define `Application\Ports`. Ninguna firma menciona `Illuminate`, `Model`, `Collection`, `Builder` ni `UploadedFile`.

| Puerto | Responsabilidad | Adaptador (Infrastructure) |
|---|---|---|
| `Clock` | `now(): DateTimeImmutable` en UTC | `SystemClock` |
| `PasswordHasher` | `hash(string)`, `verify(string, string)` | `BcryptPasswordHasher` |
| `TokenIssuer` | `issue(userId, role): IssuedToken` y `verify(string): TokenClaims` | `JwtTokenIssuer` |
| `CoverStorage` | `put(string $bytes): string` (devuelve clave), `get(key)`, `delete(key)` | `LocalCoverStorage` |
| `TransactionRunner` | `run(callable): mixed` | `DbTransactionRunner` |
| `BookRepository` | `find`, `lockByIds` (ordenados), `save`, `search`, `hasOpenLoans`, `softDelete` | `EloquentBookRepository` |
| `MemberRepository` | `find`, `save`, `search`, `existsByDocument`, `countUnreturnedLines`, `hasOverdueLines` | `EloquentMemberRepository` |
| `LoanRepository` | `save`, `find`, `findItemLocked`, `search`, `overdueLines` | `EloquentLoanRepository` |
| `UserRepository` | `findByUsername`, `save` | `EloquentUserRepository` |
| `CategoryRepository` | `all`, `find` | `EloquentCategoryRepository` |
| `ReportReader` | `mostBorrowed(DateRange): array` | `EloquentReportReader` |

Los resultados de lectura (páginas, filas de reporte) se devuelven como objetos de valor simples o arreglos planos de PHP,
nunca como paginadores de Laravel.

## 5. Servicios de aplicación (uno por caso de uso)

| Servicio | Historia |
|---|---|
| `LoginService` | HU-01 |
| `CreateLibrarianService` | HU-02 |
| `CreateBookService`, `UpdateBookService`, `DeleteBookService`, `SearchBooksService`, `GetBookService` | HU-03, HU-04 |
| `SetBookCoverService`, `RemoveBookCoverService` | HU-03 (CA-03.4) |
| `CreateMemberService`, `SearchMembersService`, `GetMemberService` | HU-05 |
| `LendBooksService` | HU-06 |
| `ReturnItemService` | HU-07 |
| `GetLoanService`, `SearchLoansService` | HU-08 |
| `OverdueService` | HU-09 |
| `MostBorrowedReportService` | HU-10 |

## 6. Punto único de composición (Art. III)

`app/Providers/PortBindingsServiceProvider.php` es el único archivo donde aparece `bind(Puerto::class, Adaptador::class)`.
Se registra en `bootstrap/providers.php`. Fuera de él:

- ningún `new` de una clase de `Infrastructure`;
- los controladores reciben servicios por inyección de dependencias.

Se verifica con un test de arquitectura ([§8](#8-pruebas-de-arquitectura-con-pest)).

## 7. Datos y migraciones

| Tema | Traducción |
|---|---|
| DDL | Solo en `database/migrations/`. `infra/` no contiene `CREATE TABLE` |
| Esquema | `CREATE SCHEMA IF NOT EXISTS library` en la primera migración; `search_path=library` en `config/database.php` |
| Tablas | Singular, con `$table` explícito: `category`, `book`, `member`, `loan`, `loan_item`, `user` |
| Sin `DEFAULT` | Ninguna columna lo lleva; los valores los pone el código |
| Sin auditoría | `$timestamps = false` y sin `created_at`/`updated_at` |
| IDs | UUID v4 con `Str::uuid()`; `$incrementing = false`; `keyType = 'string'` |
| Índices parciales y `CHECK` | `DB::statement(...)` dentro de la migración |
| Semilla | Las 5 categorías, con IDs fijos del data-model §9, en la migración inicial |
| Tabla `migrations` | La crea Laravel; es infraestructura del framework, no una tabla de negocio |
| Tablas por defecto de Laravel | `users`, `cache`, `jobs`, `sessions`, `password_reset_tokens` **no se crean**: se eliminan sus migraciones |

## 8. Pruebas de arquitectura con Pest

Se traducen los artículos I, II y III con `arch()`:

```php
arch('Art. I: el dominio no depende de nada')
    ->expect('App\Domain')
    ->not->toUse(['Illuminate', 'Eloquent', 'App\Application', 'App\Infrastructure', 'App\Http']);

arch('Art. II: la aplicación no conoce la infraestructura ni HTTP')
    ->expect('App\Application')
    ->not->toUse(['App\Infrastructure', 'App\Http', 'Illuminate\Database']);

arch('Art. III: Infrastructure solo se instancia en la composición')
    ->expect('App\Infrastructure')
    ->not->toBeUsedIn(['App\Http', 'App\Application', 'App\Domain']);
```

Complemento para "ningún `new` de infraestructura fuera de la composición": un test que recorre `app/` salvo el provider y falla si
encuentra `new App\Infrastructure\` o el nombre de un adaptador en un `new`.
El Art. I permite la biblioteca base de PHP (`DateTimeImmutable`, `InvalidArgumentException`).

## 9. Pruebas

| Nivel (plan §3) | Herramienta | Contra qué |
|---|---|---|
| Dominio | Pest | Objetos puros, sin base ni framework |
| Aplicación | Pest | Fakes de los puertos en `tests/Fakes/`. Sin tocar `Infrastructure` |
| Integración | Pest + `RefreshDatabase` | PostgreSQL real de prueba (`library_test`) |
| Contrato | Pest (`Feature`) | Cuerpo de error único y códigos de estado |
| Arquitectura | Pest `arch()` | Código fuente |
| Concurrencia | Pest (dos procesos) | PostgreSQL real; exactamente un 201 y un 409 |

Los tests se escriben en cada tarea antes de la implementación.
**Mientras no haya PHP en la máquina, los tests no se ejecutan** y `ESTADO.md` lo declara.

## 10. Validación

| Spec | Laravel |
|---|---|
| FluentValidation | Form Requests (`app/Http/Requests`) con `messages()` en español |
| Reglas de forma (longitudes, UUID, 1 a 3 líneas) | Reglas del Form Request |
| Reglas de negocio (3 máximo, vencidos, disponibles) | Dominio y servicios, con `DomainException` |
| Cuerpo de error | `failedValidation` y el handler global lo convierten a `{code, message, details}` con `VALIDATION_ERROR` |

## 11. Errores (RNF-4)

Un solo handler en `bootstrap/app.php` mapea todo al cuerpo `{ code, message, details }`:

| Excepción | `code` | Estado |
|---|---|---|
| `ValidationException` | `VALIDATION_ERROR` | 422 |
| `AuthenticationException` y fallos de token | `INVALID_CREDENTIALS` (login) o `UNAUTHENTICATED` | 401 |
| Rol insuficiente | `FORBIDDEN_ROLE` | 403 |
| `ModelNotFoundException`, `NotFoundHttpException` | `NOT_FOUND` | 404 |
| `App\Domain\Exceptions\DomainException` | el propio `code` | el propio `status` |
| Cualquier otra `Throwable` | `INTERNAL_ERROR` con mensaje genérico, sin traza | 500 |

`UNAUTHENTICATED` e `INTERNAL_ERROR` no están en el §2 del contrato. Su registro y justificación están en `decisiones.md` (D12 y D13).

## 12. Frontend (React + Vite)

| Tema | Traducción |
|---|---|
| Cliente HTTP | `fetch` con un envoltorio único que añade `Authorization: Bearer` y lee el cuerpo de error único |
| Token | En memoria; copia en `sessionStorage` para sobrevivir a recargas. Nunca en `localStorage` |
| Rutas | React Router con un componente `ProtectedRoute` que recibe los roles permitidos |
| Rol visible | Viene de `role` en la respuesta de login |
| Portadas | `GET /books/{id}/cover` exige token, así que se pide con `fetch` y se pinta con un `blob:` URL (no con `<img src>` directo) |
| Mensajes | Se muestra el `message` del error tal cual (ya viene en español) |
| CORS | El API solo admite el origen de `CORS_ORIGIN` |

## 13. Qué NO cambia

- Rutas, métodos, cuerpos y códigos de estado de `api-contract.md`.
- Columnas, tipos, restricciones e índices de `data-model.md`.
- Las 10 historias, sus criterios y las 6 reglas de negocio.
- Los 5 requisitos no funcionales.
- Lo que está fuera de alcance: multas, pagos, reservas, renovaciones, correo/SMS, importación masiva, préstamos entre sedes, atributos extra de libro.
