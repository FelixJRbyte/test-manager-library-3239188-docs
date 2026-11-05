# Constitución — Manager Library (.NET)

Principios innegociables del proyecto. No son estilo ni preferencia: cada artículo trae **cómo se verifica**, porque un principio que
nadie puede comprobar es una intención. Un cambio que viole un artículo se rechaza sin discutir el resto.

---

## Artículo I — El dominio no depende de nada

`src/Library.Domain/` no importa ningún otro módulo del proyecto ni librerías de terceros: solo la biblioteca base (`record`, `DateTimeOffset`); ni EF Core, ni ASP.NET, ni FluentValidation.

**Por qué:** es lo único que permite probar las invariantes sin base de datos, sin servidor y sin contenedor, y cambiar de ORM sin tocar reglas.
**Verificación:** NetArchTest prohíbe que el dominio dependa de `application`, `adapters` o `bootstrap`.

## Artículo II — El núcleo declara sus puertos; el mundo los implementa

La capa de aplicación solo conoce el dominio. Todo lo que necesita del exterior (repositorios, reloj, hash de claves, almacenamiento de
portadas, generación de tokens) lo expresa como un **puerto que ella misma define**. Ningún tipo de framework o de ORM aparece en la firma de un puerto.

**Verificación:** NetArchTest impide que `application` importe `adapters`. Los tests de aplicación corren con fakes y sin referenciar `adapters`.

## Artículo III — Un único punto de composición

Un solo módulo (`src/Library.Bootstrap/Composition/PortBindings.cs`) es donde un puerto se encuentra con su adaptador. Fuera de él no se instancia infraestructura.
**Verificación:** ninguna instanciación de una clase de `adapters` fuera de ese módulo; se comprueba con un test de arquitectura.

## Artículo IV — SOLID no es decorativo

| Principio | Aplicación concreta | Verificación |
|---|---|---|
| **SRP** | Un servicio de aplicación = un caso de uso. `LendBooksService` presta y no hace nada más | Un servicio con dos razones para cambiar se parte |
| **OCP** | Cambiar el almacenamiento de portadas (disco → S3) no toca dominio ni aplicación | El adaptador nuevo solo añade una línea en la composición |
| **LSP** | Los fakes de test sustituyen al adaptador real sin cambiar expectativas | El mismo test pasa contra el fake y el real |
| **ISP** | Puertos pequeños: `Clock`, `PasswordHasher`, `TokenIssuer`, `CoverStorage` | Ningún implementador deja métodos sin usar |
| **DIP** | El puerto lo define quien lo consume | Los puertos viven en el paquete de aplicación |

## Artículo V — El esquema es propiedad de las migraciones

El repositorio `infra` levanta un motor **vacío**. Todo el DDL (tablas, índices, restricciones, extensiones y semilla) vive en las migraciones de
la aplicación (migraciones de EF Core). Ver [ADR-001](adr/adr-001-propiedad-del-esquema.md).
**Verificación:** `infra` no contiene `CREATE TABLE`.

## Artículo VI — Las invariantes que la base puede imponer, la base las impone

Una regla que solo vive en código protege a la aplicación, no a los datos. Toda invariante expresable en el motor (`CHECK`, `UNIQUE`, FK)
se declara ahí. Lo que no está aplicado se marca **pendiente** en [`data-model.md`](data-model.md); no se promete.
**Verificación:** las consultas de `data-model.md` §10 coinciden con lo que declara el documento.

## Artículo VII — Lo derivado se calcula, no se almacena

El estado de un préstamo (`open`, `overdue`, `closed`) y los días de retraso se calculan. No hay columna para ellos.
**Verificación:** `data-model.md` no lista ninguna columna `status` ni `days_late`.

## Artículo VIII — Los valores congelados no siguen al catálogo

La línea de préstamo guarda título, ISBN y categoría del instante del préstamo. Ningún reporte los lee del catálogo vivo.
**Verificación:** una prueba que renombra un libro después de prestarlo y comprueba que el reporte conserva el título viejo ([ADR-004](adr/adr-004-reporte-agregado-y-congelado.md)).

## Artículo IX — Ningún secreto entra al repositorio ni a un log

Ni contraseñas, ni hashes precalculados, ni la clave del token. Todo secreto viene del entorno. El `.env` real está en `.gitignore`; solo se versiona `.env.example`.
**Verificación:** `gitleaks` o un `grep` de patrones en CI; ninguna respuesta HTTP incluye `password` ni `password_hash`.

## Artículo X — Gana el motor

Si el documento de datos y la base discrepan, el documento está roto y se corrige el documento.

## Artículo XI — Idioma

Código, columnas y rutas en inglés. Documentación y mensajes al usuario en español.

## Artículo XII — Nada se da por hecho sin evidencia

Una tarea de [`tasks.md`](tasks.md) está terminada cuando su **evidencia** existe y se puede ejecutar, no cuando «ya está programada».
