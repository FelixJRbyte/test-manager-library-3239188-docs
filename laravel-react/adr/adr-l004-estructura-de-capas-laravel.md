# ADR-L004 · Estructura de capas en Laravel

- **Estado:** aceptada
- **Adapta:** `spec-.net/architecture.md` §1-§3 y `constitution.md` Arts. I, II, III y IV.
- **Contexto:** el spec describe una arquitectura hexagonal con un proyecto por capa (`Library.Domain`, `Library.Bootstrap`, …) y la verifica con NetArchTest.
  Laravel es un solo proyecto, con una estructura propia (`app/Models`, `app/Http`) que mezcla Eloquent con el dominio. El reto dura 4 horas.
- **Qué cambió y por qué:**
  - Los proyectos se vuelven **espacios de nombres** dentro de `app/`: `Domain`, `Application`, `Infrastructure`, `Http`, `Providers`.
  - Los modelos Eloquent **no son el dominio**: son adaptadores y viven en `Infrastructure\Persistence`. Los repositorios convierten entre modelo y entidad.
  - NetArchTest se sustituye por `arch()` de Pest.
  - `PortBindings.cs` se sustituye por un único `PortBindingsServiceProvider`.
- **Decisión:**

  | Capa | Contenido | Puede depender de |
  |---|---|---|
  | `Domain` | Clases PHP puras, excepciones de dominio | Solo la biblioteca estándar de PHP |
  | `Application` | Un servicio por caso de uso y los puertos (interfaces) | `Domain` |
  | `Infrastructure` | Modelos Eloquent, repositorios, hasher, JWT, reloj, portadas, transacciones | `Application`, `Domain`, Laravel y librerías |
  | `Http` | Controllers delgados, Form Requests, API Resources, middleware | `Application`, `Domain` |
  | `Providers` | `PortBindingsServiceProvider`: el único `bind(Puerto, Adaptador)` | todas |

  Pruebas de arquitectura (Arts. I, II, III) en `tests/Arch/`. El Art. I no permite `Illuminate` ni `Eloquent` en `Domain`.
  Los controllers dependen de servicios, nunca de repositorios ni de modelos.
- **Alternativas descartadas:**
  - *Laravel "de manual" (lógica en controllers y modelos):* más rápido, pero no cumple los Arts. I a III ni permite probar con fakes.
  - *Paquetes separados con Composer (un `composer.json` por capa):* fiel al spec, pero costoso de montar sin poder ejecutar nada.
  - *Eloquent como entidad de dominio:* acopla las reglas a la base y rompe el Art. I.
- **Consecuencias:**
  - Hay que mapear modelo ↔ entidad en cada repositorio: algo de código repetido a cambio de reglas probables sin base de datos.
  - Se mantiene ligero: pocas clases, y los objetos de valor se reducen a `Isbn` y `DateRange`.
  - Los tipos de Laravel no aparecen en las firmas de los puertos. Las páginas se devuelven como arreglos planos u objetos propios, no como paginadores (que además no dan la forma `{items,page,size,total}`).
  - `Http` puede usar tipos de Laravel (`Request`, `Resource`): es un adaptador de entrada.
- **Cómo se verifica:** `tests/Arch/LayersTest.php` falla si `Domain` importa `Illuminate` o `Eloquent`, si `Application` importa `Infrastructure`, o si se hace `new` de un adaptador fuera del provider.
  Los tests de `Application` corren con fakes y no referencian `Infrastructure`.
