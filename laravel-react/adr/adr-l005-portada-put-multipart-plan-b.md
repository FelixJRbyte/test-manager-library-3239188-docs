# ADR-L005 · Portada: `PUT` multipart y plan B con `POST` + `_method`

- **Estado:** aceptada (con plan B documentado)
- **Adapta:** `spec-.net/plan.md` D-08, `api-contract.md` §3.4 y §4 (subida de portada), `data-model.md` §7.
- **Contexto:** el contrato fija `PUT /books/{id}/cover` con `multipart/form-data` y campo `file`. PHP **no puebla** `$_FILES` ni `$_POST` en una petición `PUT`
  (solo lo hace en `POST`). Laravel, en `PUT` con multipart, puede entregar `$request->file('file')` nulo.
  Ese comportamiento depende de la versión de Symfony/PHP, y no se puede comprobar en esta máquina (no hay PHP).
- **Qué cambió y por qué:** el spec .NET no tiene este problema: ASP.NET lee multipart con cualquier verbo. En Laravel el verbo importa.
- **Decisión:**
  1. **Contrato primero:** se declara `Route::put('/books/{id}/cover')` tal como pide el contrato.
  2. **Plan B, sin rutas nuevas:** el cliente React envía `POST` multipart con el campo extra `_method=PUT`.
     PHP sí puebla `$_FILES` en un `POST`, y Laravel (Symfony) sustituye el verbo por `PUT` antes de enrutar,
     así que la petición entra por la **misma** ruta `PUT` y por el mismo middleware (`admin`) y Form Request.
     No se declara ninguna ruta `POST`: el contrato de rutas queda intacto.
  3. El controlador lee `$request->file('file')`. Si llega nulo, responde 422 `VALIDATION_ERROR` con "adjunte el archivo en el campo `file`",
     nunca un 500.
  4. Se verifica con `curl -X PUT -F file=@x.png` cuando haya PHP. Si el `PUT` nativo funciona, el cliente puede usarlo directamente;
     si no, React usa `POST` + `_method=PUT`. Hasta entonces, **React usa el plan B por defecto**.
- **Validación y almacenamiento:**
  - Contenido, no extensión: `mimetypes:image/jpeg,image/png` y `max:2048` (KB).
  - Se guarda con **clave opaca**: un UUID v4 sin extensión. En `book.cover_key` va esa clave, nunca una ruta (CA-03.4).
  - Sin portada, `cover_key` es `null`. **Nunca** `""`.
  - Quitar la portada: **primero** se pone `cover_key = null` y se confirma la transacción; **después** se borra el binario (`data-model.md` §7).
    Si el borrado del binario falla, queda un archivo huérfano (inocuo), no una referencia rota.
  - Reemplazar: se guarda el binario nuevo, se actualiza `cover_key`, se confirma y se borra el binario viejo.
- **Alternativas descartadas:**
  - *Declarar una ruta `POST` propia:* añade una ruta fuera del contrato y nunca se alcanzaría con `_method=PUT`.
  - *Cambiar el contrato a `POST`:* el contrato manda; el servidor sigue exponiendo `PUT`.
  - *Aceptar el cuerpo binario crudo en `PUT` (`Content-Type: image/png`):* cambia el contrato (`multipart`, campo `file`).
  - *Parsear el multipart a mano en `PUT`:* frágil y no aporta nada si `POST` funciona.
- **Consecuencias:**
  - Ninguna ruta nueva: el servidor cumple el contrato tal cual. La diferencia vive solo en el cliente React, que envía `POST` + `_method=PUT`.
  - Un `POST` directo a `/books/{id}/cover` **sin** `_method` responde 404/405 (no existe esa ruta): es lo esperado.
  - **Pendiente de ejecución:** hasta que corra con PHP real, no se sabe si el `PUT` nativo funciona. `ESTADO.md` lo declara.
- **Cómo se verifica:** `Feature/CoverTest`: PNG de 2 MB → 200; PNG de 3 MB → 422; GIF renombrado `.png` → 422; sin portada `cover_key` es `null`;
  tras quitarla, `GET /books/{id}/cover` → 404; `PUT` y `POST` con `_method=PUT` dan el mismo resultado.
