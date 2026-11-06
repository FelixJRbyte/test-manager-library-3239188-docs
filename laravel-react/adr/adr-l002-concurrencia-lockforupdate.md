# ADR-L002 · Concurrencia con `lockForUpdate`, no con testigo de versión

- **Estado:** aceptada
- **Adapta:** `spec-.net/adr/adr-002-concurrencia-optimista.md` y `plan.md` D-04.
- **Contexto:** dos bibliotecarios pueden prestar el último ejemplar a la vez (RNF-2). El ADR-002 original usa un **testigo de versión** por libro (concurrencia optimista, como el `RowVersion`/`xmin` de EF Core).
- **Qué cambió y por qué:**
  1. `data-model.md` §3 define las columnas de `book` y **no incluye ninguna columna de versión**. Por la jerarquía (data-model manda sobre plan) y por el Art. X, no se añade una.
  2. Laravel/Eloquent no trae control de concurrencia optimista integrado. Habría que escribirlo a mano con `UPDATE … WHERE version = ?`.
  3. Postgres ofrece `SELECT … FOR UPDATE`, y Eloquent lo expone como `lockForUpdate()`.
- **Decisión:** dentro de una transacción (`TransactionRunner`):
  1. Se leen y se **bloquean** los libros del préstamo con `lockForUpdate()`, **ordenados por `id`**. El orden fijo evita interbloqueos entre dos préstamos que comparten libros en distinto orden.
  2. Se comprueba `available_copies > 0` con la fila ya bloqueada. Si no, 409 `BOOK_NOT_AVAILABLE` y la transacción se revierte entera (no se registra nada, CA-06.3).
  3. Se descuenta, se crea el préstamo con sus líneas y se confirma.
  La última barrera sigue siendo el `CHECK` `ck_book_available_range`.
  La devolución usa la misma técnica: bloquea la línea (`lockForUpdate`); si `returned_at` ya tiene valor, 409 `ITEM_ALREADY_RETURNED`.
- **Alternativas descartadas:**
  - *Testigo de versión (spec original):* exige una columna que el modelo no tiene.
  - *`UPDATE book SET available_copies = available_copies - 1 WHERE id = ? AND available_copies > 0` y comprobar filas afectadas:* válida y sin bloqueo explícito, pero obliga a mezclar el descuento con la persistencia y complica `Loan.addItem → Book.withdraw` del dominio. Queda como plan B documentado.
  - *Contador en memoria:* no sobrevive a más de un proceso.
- **Consecuencias:**
  - El argumento del ADR original contra el bloqueo ("serializa toda la biblioteca") no aplica: el bloqueo es **por fila** de libro y dura solo la transacción. Dos préstamos de libros distintos no se esperan.
  - Dos peticiones por el mismo libro se serializan: la segunda espera, lee `available_copies = 0` y recibe 409. No hace falta reintentar.
  - La transacción debe ser corta: nada de llamadas externas dentro.
  - Si el `CHECK` salta, algo escribió fuera del adaptador: es un defecto, no un caso de negocio (se conserva la lectura del ADR original).
- **Cómo se verifica:** `Feature/ConcurrencyTest` lanza dos peticiones simultáneas por un libro con un solo ejemplar: exactamente una responde 201 y la otra 409; `available_copies` queda en 0.
  `Unit/ReturnItemServiceTest` comprueba que devolver dos veces suma una sola vez.
