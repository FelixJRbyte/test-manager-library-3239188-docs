# Tareas — Manager Library (Laravel + React)

Adaptación de `spec-.net/tasks.md` (T-01..T-23) a las diez tareas del reto (T0..T9).
Cada tarea termina cuando su **evidencia** existe y se puede ejecutar (Art. XII). Una tarea se trabaja y se revisa entera antes de pasar a la siguiente.

**Estado real:** consultar [`../ESTADO.md`](../ESTADO.md). Este documento define qué se hace; ESTADO.md dice qué se hizo y si se ejecutó.

## Mapa de correspondencia con el spec .NET

| Reto | Cubre (spec .NET) |
|---|---|
| T0 | T-01, T-17 (parte), T-20 |
| T1 | T-02, T-06, T-07 (login), T-14, T-16 (errores y roles), T-17, T-18 |
| T2 | T-07 (alta de bibliotecario) |
| T3 | T-03, T-04, T-08, T-12 |
| T4 | T-06 (Member), T-12 |
| T5 | T-05, T-09, T-13 |
| T6 | T-10 |
| T7 | T-11 |
| T8 | T-22 |
| T9 | T-15, T-21, T-23 |
| Transversal | T-19 (pruebas de arquitectura): se escriben en T0 y crecen con cada capa |

## Orden de construcción

`docs` → `infra` → `api` (T0 esqueleto → T1 … T7) → `app` (T8) → `api` portada, `tool`, `page` (T9).

---

## T0 · Infra y esqueleto

- **Repos:** `infra`, `api`.
- **Cubre:** RNF-5. CA-01.3 (solo `/health` público).
- **Entregables:**
  - `infra/docker-compose.yml`: `postgres:16` vacío con volumen, contenedor `api` (Laravel), red propia. Los builds salen de `../api`.
    El servicio `app` (build de `../app`) se declara con `profiles: ["web"]` y se activa en T8, cuando `app/` ya tiene un `Dockerfile`:
    así cada commit deja el compose coherente (un build de una carpeta vacía lo rompería).
  - `infra/.env.example` con las variables del plan §2.
  - `api`: esqueleto Laravel limpio, `Dockerfile`, entrada que ejecuta `migrate --force` y `library:ensure-admin` al arrancar.
  - `GET /api/v1/health` → `{"status":"ok"}`.
  - Estructura de carpetas `Domain/Application/Infrastructure/Http` y `PortBindingsServiceProvider` vacío.
  - Prueba de arquitectura (Arts. I, II, III) y prueba de `/health`.
- **Hecho cuando:** `docker compose up` levanta todo con un comando, y `infra/` no contiene `CREATE TABLE`.
- **Evidencia:** `curl http://localhost:8000/api/v1/health` → `{"status":"ok"}`; `grep -ri "create table" infra/` sin resultados; `php artisan test --filter=Arch`.

## T1 · Migraciones, errores, login, roles, admin, categorías

- **Repo:** `api`.
- **Cubre:** HU-01 (CA-01.1, 01.2, 01.3), RNF-3, RNF-4, Arts. V, VI, VII, IX. D3, D4, D8, D9, D11, D12, D13, D14.
- **Entregables (un commit cada uno):**
  1. Migración `library`: esquema, 6 tablas, FK, `CHECK` (incluye `ck_book_total_positive` y `ck_user_role`, D9), índices (parcial de ISBN, `loan_item (due_at) WHERE returned_at IS NULL`, etc.) y las 5 categorías. Sin `DEFAULT`.
  2. Modelos Eloquent con las convenciones de D8 (en `Infrastructure/Persistence`).
  3. `DomainException` propia y handler global con el cuerpo único (RNF-4).
  4. Puertos `Clock`, `PasswordHasher`, `TokenIssuer`, `UserRepository`, `CategoryRepository` y sus adaptadores.
  5. `LoginService` y `POST /auth/login` (401 con mensaje idéntico, sin loguear el cuerpo).
  6. Middleware `JwtAuthenticate` y `RequireRole` (401 / 403).
  7. Comando `library:ensure-admin` idempotente.
  8. `GET /categories`.
- **Hecho cuando:** login devuelve `{access_token, token_type, expires_in: 28800, role}`; sin token 401; rol insuficiente 403; el cuerpo de error es siempre el mismo.
- **Evidencia:** tests `Feature/AuthTest`, `Feature/ErrorContractTest`, `Feature/SchemaTest` (consultas de data-model §10 sin `DEFAULT`, con los `CHECK`); `Unit/LoginServiceTest` con fakes.

## T2 · Usuarios

- **Repo:** `api`.
- **Cubre:** HU-02 (CA-02.1, 02.2, 02.3), RNF-3.
- **Entregables:** `User` de dominio (normaliza minúsculas y recorta), `CreateLibrarianService`, `POST /users` solo admin, Form Request.
- **Hecho cuando:** `"Ana "` queda `ana`; duplicado → 409 `USERNAME_TAKEN`; `role: admin` → 422; la respuesta nunca trae `password` ni `password_hash`.
- **Evidencia:** `Unit/UserTest`, `Unit/CreateLibrarianServiceTest` (fakes), `Feature/UsersTest` (incluye que la respuesta no contiene `password`).

## T3 · Libros

- **Repo:** `api`.
- **Cubre:** HU-03 (CA-03.1, 03.2, 03.3, 03.5), HU-04 (CA-04.1, 04.2, 04.3). D2, D5, D16, D23, D24. Trampa 9 (baja lógica).
- **Entregables:**
  - `Book` de dominio y `Isbn`.
  - `BookRepository`, servicios de crear, editar, baja, obtener y buscar.
  - `GET/POST /books`, `GET/PUT/DELETE /books/{id}`.
  - Búsqueda con `ILIKE` en título y autor, `category_id`, solo activos, orden por título, paginación construida a mano.
  - Baja lógica con `deleted_at`; 409 `BOOK_HAS_OPEN_LOANS`.
- **Hecho cuando:** ISBN de 13 dígitos único entre activos; disponibles arrancan iguales al total; cambiar el total ajusta disponibles y nunca los deja `< 0`; libro dado de baja → 404.
- **Evidencia:** `Unit/BookTest` (nunca `< 0`, nunca `> total`), `Unit/*BookServiceTest` (fakes), `Feature/BooksTest` (409, 404, paginación `{items,page,size,total}`, `size=51` → 422).

## T4 · Socios

- **Repo:** `api`.
- **Cubre:** HU-05 (CA-05.1, 05.2, 05.3). D17.
- **Entregables:** `Member`, `MemberRepository`, `CreateMemberService`, `SearchMembersService`, `GetMemberService`; `GET/POST /members`, `GET /members/{id}` (con `active_loans`). No hay `DELETE`.
- **Hecho cuando:** documento duplicado → 409 `DOCUMENT_TAKEN`; búsqueda por nombre o documento; `active_loans` cuenta líneas sin devolver; rol `librarian` permitido.
- **Evidencia:** `Unit/MemberTest`, `Feature/MembersTest` (incluye que no existe `DELETE /members/{id}` → 404/405).

## T5 · Préstamos (corazón del spec)

- **Repo:** `api`.
- **Cubre:** HU-06 (CA-06.1 a 06.7), RN-1, RN-2, RN-3, RN-4, RN-5, RNF-2. D1, D7, D22. Trampa 5.
- **Entregables:**
  - `Loan`, `LoanItem` de dominio (14 días, 1 a 3 líneas, sin repetidos, congelado).
  - `LoanRepository`, `TransactionRunner`, `LendBooksService`.
  - `POST /loans` con transacción, `lockForUpdate` sobre los libros ordenados por id, y el orden de validaciones de D1.
- **Hecho cuando:** descuenta 1 ejemplar por línea en la misma transacción; sin ejemplares → 409 y no se registra nada; `due_at = loaned_at + 14 días`; título, ISBN y categoría congelados; el registrador es el usuario autenticado.
- **Evidencia:** `Unit/LoanTest` (reloj falso), `Unit/LendBooksServiceTest` (fakes: orden D1, límite, vencidos, sin ejemplares), `Feature/LoansCreateTest`, `Feature/ConcurrencyTest` (dos peticiones por el último ejemplar: un 201 y un 409).

## T6 · Devolución, detalle, listado y vencidos

- **Repo:** `api`.
- **Cubre:** HU-07, HU-08, HU-09. D10, D18, D22. Trampas 4 y 6.
- **Entregables:** `ReturnItemService`, `GetLoanService`, `SearchLoansService`, `OverdueService`; `POST /loans/{loan_id}/items/{item_id}/return`, `GET /loans/{id}`, `GET /loans`, `GET /loans/overdue` (declarada **antes** de `/loans/{id}`).
- **Hecho cuando:** devolver pone `returned_at` y suma 1 una sola vez; repetir → 409 `ITEM_ALREADY_RETURNED`; línea de otro préstamo → 404; estado calculado; vencidos ordenados por más retraso; el filtro `status` usa `whereExists`.
- **Evidencia:** `Unit/ReturnItemServiceTest` (fakes: dos devoluciones, una suma), `Feature/ReturnTest`, `Feature/LoansListTest`, `Feature/OverdueTest`.

## T7 · Reporte

- **Repo:** `api`.
- **Cubre:** HU-10 (CA-10.1 a 10.5), Art. VIII, ADR-004. D6.
- **Entregables:** `DateRange`, puerto `ReportReader`, `MostBorrowedReportService`, `GET /reports/most-borrowed` solo admin. La consulta agrupa por `book_id, book_title, category_name` **de `loan_item`**, sin ningún `join` al catálogo.
- **Hecho cuando:** `to < from` → 422 `INVALID_DATE_RANGE`; rango vacío → 200 con `rows: []`; no hay socios ni bibliotecarios en la respuesta; `librarian` recibe 403.
- **Evidencia:** `Unit/DateRangeTest`, `Feature/ReportTest` (**renombra el libro después de prestarlo y el reporte conserva el título viejo**).

## T8 · Frontend React

- **Repo:** `app`.
- **Cubre:** T-22 del spec, y la cara visible de HU-01, 03 (consulta), 04, 05, 06, 07, 09, 10.
- **Entregables (un commit por pantalla):** esqueleto Vite y cliente `fetch` con error único; login y sesión; rutas protegidas por rol; catálogo con búsqueda y paginación; socios; nuevo préstamo; detalle y devolver línea; vencidos; reporte (solo admin).
- **Hecho cuando:** el recorrido login → buscar libro → crear socio → prestar → devolver → vencidos → reporte funciona sobre la API.
- **Evidencia:** pruebas de componentes con Vitest para el cliente y `ProtectedRoute`; recorrido manual descrito en el README.

## T9 · Portada, tool, page

- **Repos:** `api`, `tool`, `page`.
- **Cubre:** CA-03.4, RNF-1, T-15, T-21, T-23. Trampa 8.
- **Entregables:**
  - `api`: `CoverStorage`, `PUT/DELETE /books/{id}/cover`, `GET /books/{id}/cover`.
  - `tool`: sembrador por la API (no por SQL) y medición de la búsqueda con 5 000 libros.
  - `page`: HTML estático mínimo.
  - README en cada repo.
- **Hecho cuando:** `cover_key` es `null` sin portada (nunca `""`); imagen por contenido (JPG/PNG) y máx. 2 MB; la clave es un UUID opaco sin extensión; al quitar la portada primero se anula `cover_key` y se confirma, después se borra el binario.
- **Evidencia:** `Feature/CoverTest` (PNG ok, 3 MB → 422, GIF → 422); salida del medidor de `tool`; `page/index.html` abre sin la API.
- **Recortable:** si falta tiempo, `tool` y `page` quedan mínimos y se anotan en `ESTADO.md`.

---

## Reglas transversales (aplican a todas las tareas)

| Regla | Dónde se comprueba |
|---|---|
| Un solo cuerpo de error | `Feature/ErrorContractTest`, ampliado por cada tarea |
| Ningún secreto en el repo ni en logs | `grep` de patrones; `.env` ignorado |
| Paginación `{items, page, size, total}` | Cada listado |
| Fechas `Y-m-d\TH:i:s\Z` | API Resources |
| 403 no revela existencia | El recurso se busca dentro del servicio, después del middleware de rol |
| Cada commit deja el repo coherente | Mensaje `tipo(repo): Tn descripción (HU-xx)` |

## Definición de terminado

Una tarea está **hecha** cuando: (1) su código y sus tests están escritos, (2) se releyeron contra `api-contract.md` y `data-model.md`, (3) `ESTADO.md` refleja la verdad y (4) está subida.
Sin PHP en la máquina, el mejor estado alcanzable en `api` es **implementado, no ejecutado**.
