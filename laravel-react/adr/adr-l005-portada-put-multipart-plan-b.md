# ADR-L005 · Portada: `PUT` multipart y plan B con `POST` + `_method`

- **Estado:** aceptada (con plan B documentado)
- **Adapta:** `spec-.net/plan.md` D-08, `api-contract.md` §3.4 y §4 (subida de portada), `data-model.md` §7.
- **Contexto:** el contrato fija `PUT /books/{id}/cover` con `multipart/form-data` y campo `file`. PHP **no puebla** `$_FILES` ni `$_POST` en una petición `PUT`
  (solo lo hace en `POST`). Laravel, en `PUT` con multipart, puede entregar `$request->file('file')` nulo.
  Ese comportamiento depende de la versión de Symfony/PHP, y no se puede comprobar en esta máquina (no hay PHP).
- **Qué cambió y por qué:** el spec .NET no tiene este problema: ASP.NET lee multipart con cualquier verbo. En Laravel el verbo importa.
- **Decisión:**
  1. **Contrato primero:** se declara `Route::put('/books/{id}/cover')` tal como pide el contrato.
  2. **Plan B, activo desde ya:** se declara **además** `Route::post('/books/{id}/cover')` que apunta al mismo controlador.
     El cliente React envía `POST` con el campo extra `_method=PUT` (Laravel lo acepta y lo trata como `PUT`).
     Ambas rutas comparten el mismo middleware (`admin`) y el mismo Form Request.
  3. El controlador lee `$request->file('file')`. Si llega nulo en la ruta `PUT`, responde 422 `VALIDATION_ERROR` con "adjunte el archivo en el campo `file`",
     nunca un 500.
  4. Se verifica con `curl -X PUT -F file=@x.png` cuando haya PHP. Si funciona, el plan B queda como alias inocuo.
- **Validación y almacenamiento:**
  - Contenido, no extensión: `mimetypes:image/jpeg,image/png` y `max:2048` (KB).
  - Se guarda con **clave opaca**: un UUID v4 sin extensión. En `book.cover_key` va esa clave, nunca una ruta (CA-03.4).
  - Sin portada, `cover_key` es `null`. **Nunca** `""`.
  - Quitar la portada: **primero** se pone `cover_key = null` y se confirma la transacción; **después** se borra el binario (`data-model.md` §7).
    Si el borrado del binario falla, queda un archivo huérfano (inocuo), no una referencia rota.
  - Reemplazar: se guarda el binario nuevo, se actualiza `cover_key`, se confirma y se borra el binario viejo.
- **Alternativas descartadas:**
  - *Solo `POST` con `_method`:* contradice el contrato (`PUT`).
  - *Aceptar el cuerpo binario crudo en `PUT` (`Content-Type: image/png`):* cambia el contrato (`multipart`, campo `file`).
  - *Parsear el multipart a mano en `PUT`:* frágil y no aporta nada si `POST` funciona.
- **Consecuencias:**
  - Una ruta extra fuera del contrato (`POST /books/{id}/cover`). Se documenta aquí y en el README de `api`; no cambia ninguna respuesta.
  - La `POST` deja de ser "solo `GET`/`PUT`/`DELETE`" para esta ruta; los clientes que sigan el contrato al pie de la letra no se ven afectados.
  - **Pendiente de ejecución:** hasta que corra con PHP real, no se sabe si el `PUT` nativo funciona. `ESTADO.md` lo declara.
- **Cómo se verifica:** `Feature/CoverTest`: PNG de 2 MB → 200; PNG de 3 MB → 422; GIF renombrado `.png` → 422; sin portada `cover_key` es `null`;
  tras quitarla, `GET /books/{id}/cover` → 404; ambas rutas (`PUT` y `POST` con `_method`) dan el mismo resultado.
