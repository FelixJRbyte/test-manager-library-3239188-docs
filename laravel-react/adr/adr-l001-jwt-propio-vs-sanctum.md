# ADR-L001 · JWT propio con `firebase/php-jwt`, no Sanctum

- **Estado:** aceptada
- **Adapta:** `spec-.net/plan.md` D-11 y `api-contract.md` §4.
- **Contexto:** el contrato exige "JWT firmado, 8 horas, con `sub` y `role`" y que el cliente reciba `{access_token, token_type, expires_in, role}`.
  El spec .NET lo resuelve con la librería JWT de ASP.NET. En Laravel, la opción habitual es Sanctum.
- **Qué cambió y por qué:** Sanctum **no emite JWT**: genera tokens opacos que se guardan en una tabla `personal_access_tokens`.
  Eso rompe dos cosas del spec: (1) el token no lleva `sub` ni `role`, y (2) exige una séptima tabla que `data-model.md` no tiene (Art. V y X).
- **Decisión:** `firebase/php-jwt`, algoritmo HS256, detrás del puerto `TokenIssuer` (Art. II). Un middleware propio `JwtAuthenticate` lee el encabezado
  `Authorization: Bearer`, valida firma y vigencia, y deja `user_id` y `role` en la petición. `RequireRole` decide el 403.
  Claims: `sub`, `role`, `iat`, `exp = iat + 28800`. Secreto en `JWT_SECRET`, con al menos 32 caracteres.
- **Alternativas descartadas:**
  - *Sanctum:* ver arriba; no es JWT y añade una tabla.
  - *`tymon/jwt-auth`:* resuelve lo mismo, pero trae su propio guard y configuración; más superficie para un contrato tan pequeño.
  - *Passport (OAuth2):* desproporcionado para dos roles.
- **Consecuencias:**
  - Sin estado: no hay revocación. Un token robado vale hasta que vence (8 h). Aceptable para el alcance; el spec no pide cierre de sesión en servidor.
  - Se pierde `Auth::user()`. El middleware carga el usuario por `sub` solo cuando un caso de uso lo necesita (p. ej. registrar `loaned_by_user_id`).
  - Se crea y se mantiene el middleware: poco código, pero propio.
  - Un fallo de token responde 401 `UNAUTHENTICATED` (ver D12 en `decisiones.md`).
- **Cómo se verifica:** `Feature/AuthTest` decodifica el token y comprueba `sub`, `role` y `exp - iat = 28800`. Un token manipulado o vencido da 401.
  El `grep` del repo no encuentra ningún secreto real.
