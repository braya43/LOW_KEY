# Guía para aprender a automatizar tests (desde cero)

> **Para quién es esto:** para vos, QA que ya conoce el producto (AIO/AMI, Respel, Green, Zona de Clientes) pero que **nunca escribió un test automatizado** y quiere aprender a hacerlo **solo/a**, sin depender de la IA.
>
> **La meta:** que al terminar puedas mirar un requerimiento, explorar la vista, escribir el POM, escribir el spec, correrlo, leer el rojo y arreglarlo — entendiendo **qué** hacés y **por qué**.
>
> **Cómo usar esta guía:** leela de arriba a abajo la primera vez. Después volvé a la sección que necesites como referencia. Todos los ejemplos son **reales de este repo** (módulo *Zona de Clientes › Respel › Solicitudes de Servicio*), no inventados. Si abrís esos archivos en paralelo mientras leés, se te va a fijar mucho mejor.

---

## Índice
1. [Qué es automatizar y por qué (Playwright + POM)](#1-qué-es-automatizar-y-por-qué)
2. [Mapa de la carpeta: dónde vive cada cosa](#2-mapa-de-la-carpeta)
3. [Qué es un POM y por qué no ponemos selectores sueltos en el test](#3-qué-es-un-pom-page-object-model)
4. [Cómo se escribe un test paso a paso (la receta)](#4-cómo-se-escribe-un-test-paso-a-paso)
5. [Selectores robustos y esperas deterministas](#5-selectores-robustos-y-esperas-deterministas)
6. [Las SEMILLAS (seed / precondición)](#6-las-semillas-seed--precondición)
7. [Cómo correr los tests y leer los resultados](#7-cómo-correr-y-leer-resultados)
8. [Errores típicos de principiante + las reglas de oro del repo](#8-errores-típicos--reglas-de-oro)
9. [Mini-ejercicio final para practicar solo/a](#9-mini-ejercicio-final)

---

## 1. Qué es automatizar y por qué

### La idea en una frase
**Automatizar un test = escribir un programita que abre la aplicación, hace clics y escribe como lo haría un usuario, y después COMPRUEBA que pasó lo que tenía que pasar.**

**Analogía:** imaginá que le dejás por escrito a un robot pasante *exactamente* qué botones tocar, qué escribir y qué revisar al final. La primera vez cuesta escribir las instrucciones. Pero después el robot repite esa prueba **100 veces sin cansarse, sin distraerse y en segundos**. Vos, como QA manual, harías esa misma prueba una vez y ya estarías agotado/a al décimo intento.

### Por qué automatizar (el "para qué" antes del "cómo")
- **Regresión barata:** cada vez que el dev toca algo, corrés la suite y en minutos sabés si **rompió** algo que antes funcionaba. A mano eso sería reprobar TODO el módulo cada vez.
- **No se me olvida ningún caso:** el test siempre revisa lo mismo, con la misma rigurosidad. Un humano cansado se saltea el caso 14.
- **Evidencia:** cada corrida deja screenshots, videos y logs. "Funciona" deja de ser tu palabra: es un reporte.
- **Caza bugs sutiles:** el test puede **intentar cosas que un humano no probaría** (teclear en un campo "solo lectura", mandar cantidad negativa, subir un archivo prohibido). Ahí es donde se esconden los bugs de verdad.

### Las dos herramientas que vas a oír todo el tiempo

**Playwright.**
- **En simple:** es la librería (un conjunto de herramientas de código ya hechas) que **maneja el navegador por vos**. Le decís "andá a esta URL", "hacé clic en este botón", "escribí esto", "esperá a que aparezca este texto" y Playwright lo hace en un Chrome real.
- **Para qué:** es el "robot pasante" que mueve el mouse y el teclado.
- **Ejemplo:** `await page.getByRole('button', { name: 'Guardar' }).click();` → *hacé clic en el botón que dice "Guardar"*.
- **En nuestro proyecto:** todos los tests de AIO (web) corren con Playwright sobre Chromium. (Los de AMI, la app móvil, usan otra herramienta llamada Appium, pero eso es otro mundo — no lo necesitás ahora.)

**POM (Page Object Model).**
- **En simple:** es un **patrón de organización**. En vez de mezclar los selectores (las "direcciones" de cada botón/campo) dentro del test, los guardás en una clase aparte que representa *la pantalla*. El test le pide acciones a esa clase ("creá una solicitud", "anulá la SS 42") sin saber en qué botón hay que hacer clic.
- **Para qué:** para que el test se lea como una historia de negocio, y para que cuando la web cambie un botón lo arregles **en un solo lugar** y no en 18 tests.
- Lo vemos en detalle en la [sección 3](#3-qué-es-un-pom-page-object-model).

> **TypeScript**, de paso: es el lenguaje en que está escrito todo esto. Es JavaScript "con tipos" (te avisa si te equivocás pasando un número donde iba texto). No necesitás dominarlo para empezar; se aprende leyendo los ejemplos del repo. El `await` que ves en todos lados significa "esperá a que esto termine antes de seguir" (porque manejar un navegador toma tiempo real).

---

## 2. Mapa de la carpeta

Antes de escribir nada, tenés que saber **dónde vive cada cosa**. Este repo NO es un montón de archivos sueltos: cada carpeta tiene un trabajo. Entender esto te ahorra el 80% de la confusión inicial.

### El primer corte: 3 aplicaciones bajo prueba
```
testing-aio/
├── aio/     ← tests de la WEB (AIO). Playwright + Chrome.  ← acá vas a vivir vos
├── ami/     ← tests de la APP MÓVIL (AMI). Appium. Otro mundo, ignoralo por ahora.
├── e2e/     ← tests que cruzan web + móvil en un mismo flujo. Avanzado, más adelante.
└── docs/    ← toda la documentación (el "cerebro" y los casos de prueba)
```
**Por qué están separados:** la web y el móvil se manejan con herramientas distintas (una mueve un navegador, la otra un celular). Mezclarlas sería un lío. Vos vas a trabajar dentro de `aio/`.

### Dentro de `aio/`: el corazón del asunto
```
aio/
├── pages/       ← los POMs (una clase por pantalla). El "cómo" tocar la UI.
│   ├── base/           BasePage: la clase madre que todos los POM heredan.
│   ├── components/      piezas COMUNES reutilizadas por muchas pantallas
│   │   ├── ComponentesComunes.ts   ← fachada: navegación, botones, tabla, toasts…
│   │   └── comunes/                 ← esas piezas por contexto (Navegacion, Formulario…)
│   └── modules/         ← un POM por módulo de negocio
│       ├── green/comercial/SolicitudServiciosPage.ts        (POM admin)
│       └── zona-clientes/SolicitudServiciosClientePage.ts   (POM cliente ← nuestro ejemplo)
│
├── tests/       ← los SPECS (los tests en sí). El "qué" validar.
│   ├── modules/zona-clientes/ClientesSolicitudServicio.spec.ts   ← nuestro ejemplo, 18 casos
│   ├── auth.setup.ts            ← login admin (corre 1 vez antes de todo)
│   ├── auth-cliente.setup.ts    ← login del portal cliente
│   └── seed.setup.ts            ← siembra data por API (1 vez antes de todo)
│
├── services/    ← "motores" de bajo nivel (no son tests, son herramientas)
│   ├── api/Apirequest.ts            cliente HTTP para hablar con el backend directo
│   ├── auth/AuthSessionService.ts   guarda/reinyecta la sesión (login)
│   └── data/DataService.ts          crea data de precondición por API (semillas)
│
└── ...
data/            ← constantes y "recetas" de datos (nombres de módulos, tipos de archivo, semillas)
config.ts        ← credenciales y URLs del ambiente
playwright.config.ts  ← la configuración maestra (qué se corre, en qué orden)
```

### Dentro de `docs/`: la documentación (que NO es opcional)
```
docs/
├── context/          ← el "CEREBRO": lo técnico y verificado de cada módulo
│   └── aio/zona-de-clientes/respel/solicitud-de-servicios.md   ← nuestro ejemplo
├── casos-de-prueba/  ← los casos en lenguaje de NEGOCIO (para personas, no código)
└── proceso-automatizar-modulo.md   ← el proceso oficial de 10 pasos
```

### Las 3 fuentes de información (esto es clave, memorizalo)
Cuando no sabés cómo *debería* comportarse algo, tenés 3 lugares donde mirar, y cada uno responde una **pregunta distinta**:

| Fuente | Responde | Dónde |
|---|---|---|
| **Manual de usuario** | Qué **DEBERÍA** hacer (la intención de negocio). Puede estar viejo. | `services.datint.co/Manual/` |
| **Cerebro** (`docs/context/`) | Qué **HACE de verdad** + el cómo técnico, ya verificado por nosotros (selectores, endpoints, semillas, gotchas). | `docs/context/<app>/<área>/<módulo>.md` |
| **Código** (`aio-app`, `aio-backend`) | La verdad definitiva de qué hace HOY. Es de solo lectura. | Repos aparte en `C:\TRABAJO\` |

**Regla de oro sobre las fuentes:** ninguna de las 3 reemplaza **verificar EN VIVO**. Las 3 son de lectura; el veredicto se da **ejerciendo la UI** con tus propios ojos.

> **Por qué separamos "cerebro" de "casos de prueba":** el **cerebro** es para el que va a *escribir código* (necesita el selector exacto, el endpoint, la semilla). Los **casos de prueba** son para una persona de negocio que quiere entender *qué* valida cada caso, sin ver una línea de código. Misma prueba, dos idiomas.

---

## 3. Qué es un POM (Page Object Model)

### El problema que resuelve
Imaginá que escribís esto **dentro del test**, directo:

```ts
// ❌ MAL: selectores sueltos dentro del test
await page.locator('input[name="quotation_id"]').fill('39');
await page.locator('input[name="delivery_type"]').click();
await page.getByText('Planta').click();
await page.locator('textarea[name="description"]').fill('Mi solicitud');
// ...30 líneas más de clics...
```

Problemas:
1. **Si la web cambia** `quotation_id` por `quote_id`, tenés que buscar y arreglar eso en **los 18 tests** que lo usan.
2. El test se lee como jeroglíficos: nadie entiende que eso "crea una solicitud".
3. Copiás y pegás los mismos 30 clics en cada caso.

### La solución: el POM
Un **Page Object** es una clase que representa **una pantalla**. Adentro guarda:
- los **selectores** (getters "privados", que solo usa la clase),
- los **métodos** (acciones de negocio públicas que el test llama: `crearPlanta(...)`, `anular(id)`).

El test entonces se lee así:
```ts
// ✅ BIEN: el test habla de NEGOCIO, no de selectores
const ss = new SolicitudServiciosClientePage(page);
const resp = await ss.crearPlanta(datos, residuo, entrega);
```
Se lee casi como español: *"con la página de solicitudes, creá una con entrega planta"*. El **cómo** (qué input, qué botón) está escondido dentro del POM.

### Un getter real del POM cliente, explicado línea por línea
De `aio/pages/modules/zona-clientes/SolicitudServiciosClientePage.ts`:

```ts
/** Textarea "Descripción" (`name=description`). */
private get descripcionField(): Locator {
  return this.page.locator('textarea[name="description"]').first();
}
```
- `private get descripcionField()` → es un **getter privado**: una "propiedad calculada" que solo la propia clase usa (el `private` = de puertas para adentro). Cada vez que lo pedís, te devuelve el localizador fresco.
- `: Locator` → el tipo que devuelve: un **Locator** de Playwright, que es "una dirección hacia un elemento de la página" (todavía no lo tocó, solo sabe dónde está).
- `this.page.locator('textarea[name="description"]')` → buscá en la página un `<textarea>` cuyo atributo `name` sea `description`. (`this.page` es el navegador que le pasaron al POM.)
- `.first()` → si hubiera varios que matchean, quedate con el primero (evita el error de "encontré 2, no sé cuál").
- El comentario `/** ... */` de arriba es un **docblock**, obligatorio en este repo: explica qué es en una línea.

### Un método real del POM cliente, explicado
```ts
/** Llena la pestaña 1 "Datos Básicos" SALTEANDO el Cliente (precargado). Reusa `selectCotizacion` del admin.
 *  @param opts - cotización, tipo de entrega, descripción, jornada y sucursales. @returns nada. */
async fillDatosBasicos(opts: DatosBasicosClienteSS): Promise<void> {
  await this.ss.selectCotizacion(opts.cotizacionNumero);
  await this.selectPrecargable(this.tipoEntregaField, opts.tipoEntrega);
  if (opts.jornada) await this.selectPrecargable(this.jornadaField, opts.jornada);
  for (const sucursal of opts.sucursales ?? []) {
    await this.comunes.selects.selectMultiple(this.sucursalesField, sucursal);
  }
  await this.descripcionField.fill(opts.descripcion);
}
```
- `async ... Promise<void>` → es un método asíncrono (usa `await` adentro) que no devuelve nada (`void`).
- Recibe un objeto `opts` con todos los datos de la pestaña 1. Adentro elige la cotización, el tipo de entrega, la jornada (si vino), agrega las sucursales (si hay) y escribe la descripción.
- Fijate que **usa otros getters/métodos** (`this.tipoEntregaField`, `this.descripcionField`, `this.selectPrecargable(...)`). El método es una "receta" que combina las piezas.

### El patrón de COMPOSICIÓN (nuestro caso estrella)
Acá está la joya de este módulo, y es un concepto que te va a servir muchísimo.

**El descubrimiento:** el portal del **cliente** usa **el MISMO wizard** que el módulo **admin** (Green › Comercial › Solicitudes de Servicio). Mismas 4 pestañas, mismos selectores, mismos botones. Solo cambian: la navegación al módulo y que el "Cliente" viene precargado.

**La tentación (mala):** copiar y pegar todo el POM admin en el POM cliente. Resultado: dos archivos gigantes casi idénticos; un bug arreglado en uno sigue vivo en el otro.

**La solución (buena) = COMPOSICIÓN:** en vez de copiar, el POM cliente **tiene adentro** una instancia del POM admin y le **delega** el trabajo repetido, sin tocarlo.

```ts
export class SolicitudServiciosClientePage extends BasePage {
  private comunes = new ComponentesComunes(this.page);
  /** POM admin reusado (mismo `page`): sus métodos PÚBLICOS manejan el wizard idéntico. */
  private ss = new SolicitudServiciosPage(this.page);   // ← ACÁ está la composición
```
- `private ss = new SolicitudServiciosPage(this.page)` → el POM cliente crea **su propio** POM admin, pasándole el **mismo navegador** (`this.page`). Ahora `this.ss` sabe manejar todo el wizard.
- Cuando el cliente necesita agregar un residuo, no reinventa nada: `await this.ss.agregarResiduo(residuo)`. Está usando el método del admin.
- Solo **reescribe lo que es distinto**: la pestaña 1 (porque el Cliente viene precargado) y agrega un método `anular()` propio (porque el cliente anula sin pedir motivo).

**Analogía:** en vez de construir un auto nuevo desde cero, le ponés a tu auto un motor que ya funciona (el del admin) y solo le cambiás el tablero (la pantalla de entrada). Si el motor mejora, tu auto mejora gratis.

> **La regla del repo detrás de esto:** *"No borrar ni modificar lo ajeno. Reusar código pre-existente por composición, sin tocarlo."* El POM admin quedó con **0 cambios** (`git diff` limpio) y sin embargo el cliente reusa todo. Eso es composición bien hecha.

`extends BasePage`, de paso, es **herencia**: el POM cliente hereda utilidades comunes de una clase madre `BasePage` (por ejemplo `this.page`). Herencia = "soy un tipo de…". Composición = "tengo un…". Acá usamos las dos: *es* una BasePage y *tiene* un POM admin.

---

## 4. Cómo se escribe un test paso a paso

Esta es **la receta que podés seguir sola/o** para cualquier módulo. La escribo en el orden real del proceso doc-first del repo (`docs/proceso-automatizar-modulo.md`), pero simplificada para vos.

### La regla madre: DOC-FIRST
> **No se escribe una sola línea de test hasta que exploraste la vista y documentaste lo que viste.** El código es el ÚLTIMO paso, no el primero.

¿Por qué? Porque el error #1 (el que más tiempo hace perder) es escribir código de una pantalla **que nunca miraste**, inventando selectores "de cómo suele ser". Después el test se cae en la corrida y perdiste una hora. **Mapear primero es la regla que más tiempo ahorra.**

### Paso (a) — Leé el requerimiento y el cerebro
1. Buscá la **HU** (Historia de Usuario) del requerimiento. En este repo viven como PDF en una carpeta Mantis fuera del repo. Se leen con `pdftotext -layout archivo.pdf` antes de tocar nada.
2. Leé el **cerebro** del módulo si ya existe: `docs/context/aio/zona-de-clientes/respel/solicitud-de-servicios.md`. Ahí está TODO lo que ya sabemos: qué es el módulo, las diferencias vs admin, los datos-semilla reales, los endpoints, los bugs conocidos, los gotchas.

Mirá este fragmento del cerebro (te ahorra HORAS):
> *"Cotizaciones (2, doble-aprobadas): `39 - BOGOTA - Diaria` → `quotation_id 9287`… ⚠️ **el nº visible ≠ el id interno**."*

Eso te dice que en el listado ves "39" pero por dentro el id es 9287. Si no lo supieras, tu test fallaría misteriosamente. El cerebro ya lo verificó por vos.

### Paso (b) — Explorá la vista EN VIVO para sacar selectores REALES
**Esto es innegociable.** Abrís la pantalla de verdad, con tus ojos, y capturás:
- el **selector exacto** de cada campo/botón (su `name`, su rol, su texto),
- qué **endpoint** dispara cada acción (mirando la pestaña Red del navegador),
- el **comportamiento real** (¿el combo trae lo que debería?, ¿el botón "Ver" deja editar?).

**Nunca asumas "seguro el input se llama `description`".** Miralo. En este repo, un desarrollador con IA usa herramientas de exploración (Playwright MCP); vos, a mano, usás las **DevTools del navegador** (F12 → inspeccionar elemento para ver el `name`, pestaña Network para ver el endpoint). El principio es el mismo: **evidencia real, no suposición.**

### Paso (c) — Escribí (o extendé) el POM
Con los selectores reales en mano, agregás al POM los getters y métodos que te falten. Si el POM ya existe (como el nuestro), quizás solo agregás un método nuevo. Recordá: **selectores robustos** (sección 5) y **no tocar lo ajeno** (composición).

### Paso (d) — Escribí el spec
Un spec tiene una **estructura fija**. Vamos a desarmar la de nuestro archivo real.

```ts
test.describe('Pruebas de regresion de Zona de Clientes - Respel - Solicitudes de Servicio', () => {
  // ...
  test.beforeEach(async ({ page }) => {
    await authCliente.enterConSesion(page);   // reinyecta la sesión (login ya hecho)
    const ss = new SolicitudServiciosClientePage(page);
    await ingresarConReintento(page, async () => {
      await ss.selectEmpresa();
      await ss.selectModulo(TestModulos.lista.ZONACLIENTES);
      await ss.open();
    });
  });

  test('01-Crear una SS con entrega Planta queda en estado Solicitada y Externa', async ({ page }, testInfo) => {
    // ... el caso ...
  });

  test('02-Crear una SS con recoleccion Ruta queda en estado Solicitada', async ({ page }, testInfo) => {
    // ... otro caso ...
  });
});
```

Las 3 piezas que SIEMPRE vas a ver:
- **`test.describe('...', () => { ... })`** → una **caja** que agrupa los tests de un mismo módulo. El texto es el título del grupo. Todo lo de adentro comparte configuración.
- **`test.beforeEach(async ({ page }) => { ... })`** → **"antes de CADA test, hacé esto"**. Acá se prepara el terreno: reinyecta la sesión y entra al módulo. Así cada test arranca desde el listado, sin repetir esos pasos 18 veces. El `{ page }` que recibe es el navegador fresco que Playwright te da para ese test.
- **`test('nombre', async ({ page }, testInfo) => { ... })`** → **un caso**. El nombre empieza con número (`01-`, `02-`) para poder correr uno solo con filtro. Adentro va la prueba.

### Paso (e) — Desarmemos un caso simple: el **01 (crear)**
```ts
test('01-Crear una SS con entrega Planta queda en estado Solicitada y Externa', async ({ page }, testInfo) => {
  const ss = new SolicitudServiciosClientePage(page);         // 1. instancio el POM
  const api = new ApiRequest();                                // 2. cliente HTTP para verificar por atrás
  api.setToken(authCliente.readTokenApi());                    //    con el token del cliente
  const descripcion = `SS Planta QA ${Date.now()}`;            // 3. dato ÚNICO (timestamp) → mi propia data
  let id = 0;
  try {
    const datos: DatosBasicosClienteSS = { cotizacionNumero: 39, tipoEntrega: 'Planta', descripcion };
    const resp = await ss.crearPlanta(datos, residuoGrasa('5'), entregaPlanta);   // 4. ACCIÓN por UI

    // 5. ASERCIONES: comprobar que pasó lo esperado
    expect(resp, 'debería dispararse el POST store-custom').not.toBeNull();
    expect([200, 201], `store-custom debería responder 200/201 (respondió ${resp?.status()})`).toContain(resp!.status());
    const creada = (await resp!.json()) as { data?: { id?: number } };
    id = creada.data?.id ?? 0;
    expect(id, 'la creación debería devolver el id de la SS').toBeTruthy();

    // 6. VERIFICAR POR OTRA VÍA (API): que nació "Solicitada" y "Externa"
    const show = await api.get<{ data: { status_service: string; internal: boolean } }>(`/api/service-requests/v0/show/${id}`);
    expect(show.data.status_service, 'la SS debería nacer en "Solicitada"').toBe('Solicitada');
    expect(show.data.internal, 'una SS creada por el CLIENTE debería ser "Externa" (internal=false)').toBe(false);

    await attachPageScreenshot(testInfo, page, 'SolicitudServicios-01-Planta');   // 7. evidencia
  } finally {
    await api.dispose();                                        // 8. limpieza (cerrar el cliente HTTP)
  }
});
```
La anatomía universal de un test es **AAA**: **Arrange** (preparo: pasos 1-3), **Act** (actúo: paso 4), **Assert** (compruebo: pasos 5-6). Detallado:
- `expect(algo, 'mensaje').toBe(x)` → la **aserción**: "espero que `algo` sea `x`". Si no lo es, el test **falla** y muestra el mensaje. El mensaje explica *qué esperabas*, para que el rojo se entienda sin adivinar.
- `.not.toBeNull()`, `.toContain(...)`, `.toBeTruthy()`, `.toBe(...)` → distintos tipos de comprobación (que no sea nulo / que la lista lo contenga / que tenga un valor "verdadero" / que sea exactamente igual).
- `Date.now()` en la descripción → **cada test crea su propia data única.** Nunca reusés un registro de QA que ya existe (memoria del repo: *"cada test crea su propia data"*).
- `try { ... } finally { ... }` → el `finally` corre **siempre**, falle o no el test. Se usa para **limpiar** (cerrar conexiones, borrar data de apoyo).

### Paso (e-bis) — Un caso de **validación negativa**: el **16**
Los casos que más bugs cazan son los **negativos**: probar lo que NO debería dejar hacer.
```ts
test('16-Crear sin los campos obligatorios bloquea: pestaña 1 y submodal Residuo (HU 1.4)', async ({ page }, testInfo) => {
  const ss = new SolicitudServiciosClientePage(page);
  await ss.openAgregar();
  // (a) Intentar avanzar la pestaña 1 SIN llenar nada → debe bloquear
  await ss.wizard.clickSiguiente();
  await ss.wizard.expectMensajeRequerido();                    // aparece "Este campo es requerido"
  await expect(
    page.locator('.v-overlay--active input[name="quotation_id"]'),
    'el wizard NO debería avanzar de la pestaña 1 sin los obligatorios',
  ).toBeVisible({ timeout: 10_000 });                          // sigo en la pestaña 1 (no avanzó)
  await expect(
    page.getByRole('columnheader', { name: 'Nombre Residuo' }),
    'no debería haber llegado a la pestaña "Residuos"',
  ).toHaveCount(0);                                            // NO llegué a la pestaña 2
  // ... (b) el submodal Residuo también exige sus obligatorios ...
});
```
Fijate la lógica: hago una acción "prohibida" (avanzar sin datos) y **compruebo que el sistema me frenó** — que apareció el mensaje de requerido Y que **no** avancé. Un `.toHaveCount(0)` verifica que algo **no está**. Probar la ausencia es tan importante como probar la presencia.

### Paso (f) — Correlo y leé el fallo
Ver [sección 7](#7-cómo-correr-y-leer-resultados). La regla clave cuando falla: **NO edites de oído.** Primero **explorá en vivo el paso que falló** (abrí la pantalla y mirá qué pasa de verdad). El error de consola puede mentirte; la causa real suele estar antes.

---

## 5. Selectores robustos y esperas deterministas

Un test que "a veces pasa y a veces no" (le decimos **flaky**, escamoso) es peor que ningún test: te hace desconfiar de todo. Las dos causas #1 de flakiness son **selectores frágiles** y **esperas mal hechas**. Acá está cómo evitarlas.

### Selectores: buscá por lo que el USUARIO ve, no por posición
**Bueno (estable):** por `name`, por rol, por texto, por placeholder.
```ts
page.locator('input[name="quotation_id"]')                    // por atributo name (estable)
page.getByRole('button', { name: 'Buscar', exact: true })     // por rol + texto visible
page.getByRole('columnheader', { name: 'Nombre Residuo' })    // el encabezado de columna
```
**Malo (frágil):**
```ts
page.locator('.v-list-item:nth-child(3)')       // ❌ por posición: si agregan una fila, se rompe
page.locator('#app-text-field-nombre-sgov2')    // ❌ id autogenerado: cambia por sesión, nunca matchea
```
- **Por qué no por índice:** si la web agrega una columna "Acciones" o reordena, tu `(...//td)[5]` apunta a otra cosa. Por eso el repo prohíbe índices (ver `CONVENCIONES.md`).
- **Por qué no ids autogenerados:** los `#app-text-field-...-sgov2` cambian en cada sesión → nunca van a matchear en una corrida nueva.

### El `.or()`: un plan B por si el primer selector no está
```ts
const enResiduos = this.residuoAgregarButton
  .or(this.page.getByRole('columnheader', { name: 'Nombre Residuo' }))
  .first();
```
`.or()` dice: *"buscá esto; si no, buscá esto otro"*. Sirve cuando un mismo estado se puede detectar por dos señales distintas (el botón "Agregar residuo" **o** el encabezado "Nombre Residuo" — cualquiera me confirma que llegué a la pestaña Residuos). Hace el test más resistente a variaciones.

### Esperas: esperá por EFECTO, nunca por tiempo fijo
La regla más importante de robustez del repo: **prohibido `pause` ciego** (`waitForTimeout(3000)`). ¿Por qué? En QA lento esos 3 segundos no alcanzan (falla); en QA rápido sobran (perdés tiempo). El número mágico nunca es el correcto.

En su lugar, esperá **el efecto determinista** de lo que pediste:

**1. Esperar una respuesta del servidor (`waitForResponse`):**
```ts
async open(): Promise<void> {
  await this.comunes.navegacion.enterRespelMenu();
  const listaCargada = this.page
    .waitForResponse(
      (r) => r.request().method() === 'GET' && r.url().includes('service-requests/v0/get-all-custom'),
      { timeout: 30_000 },
    )
    .catch(() => null);
  await this.comunes.navegacion.enterSolicitudesServicioMenu();
  await listaCargada;   // ← no sigo hasta que el listado REALMENTE cargó del backend
}
```
Esto dice: *"entrá al menú y no sigas hasta que llegue la respuesta del endpoint que trae el listado"*. Determinista: espera exactamente lo que tiene que pasar, ni un ms de más.

**2. Esperar a que algo sea visible (`expect(...).toBeVisible`):**
```ts
await expect(this.tituloField, 'debería verse el título "Solicitudes de Servicio"').toBeVisible({ timeout: 20_000 });
```
El `expect` con `toBeVisible` **reintenta solo** hasta que aparezca (o se acabe el timeout). No es una pausa fija: si aparece en 200ms, sigue en 200ms.

**3. Esperar a que se vayan los spinners (loaders):** todo formulario de AIO abre con "esqueletos" de carga (`.v-skeleton-loader`) mientras trae los catálogos. Si tocás un campo con el esqueleto puesto, leés vacío o tipeás al aire. Por eso hay un helper `waitLoadingGone()` que espera a que **todos** los loaders desaparezcan. Está encapsulado dentro de los métodos `open…`/`goToTab…`, así no te tenés que acordar.

> **La lección de fondo:** un test rápido le pega al botón *antes* de que la UI esté lista. Explorando a mano nunca lo reproducís (vos sos lento comparado con el código). Por eso la robustez se **diseña**, no se confía en que "se vio bien".

---

## 6. Las SEMILLAS (seed / precondición)

Esta es la parte que más te interesa aprender, así que vamos despacio y con ejemplos reales.

### El problema: un test necesita que algo YA exista
Pensá el caso 04: *"el combo Residuo NO debe ofrecer un residuo en estado Solicitado (no aprobado)"*. Para probar eso, **necesito que exista un residuo en estado Solicitado** antes de abrir el combo. Si no existe, no tengo nada que verificar.

A ese "algo que tiene que existir antes de que el test empiece" se le llama **precondición**. Y **sembrar** (seed) es **crear esa precondición**.

- **Semilla / seed:** datos de apoyo que un test necesita que existan de antemano.
- **En simple:** es "preparar el escenario" antes de que empiece la obra. Antes de probar que puedo anular una solicitud, tengo que **tener** una solicitud.
- **Ejemplo:** para probar el login necesito un usuario creado. Para probar "editar cliente" necesito un cliente. Para el caso 04, un residuo no-aprobado.
- **En nuestro proyecto:** el cliente-semilla (usuario 1794) ya tiene 2 cotizaciones aprobadas, 26 residuos y 5 sucursales. Esa precondición pesada **ya está sembrada**; los tests solo siembran lo puntual que les falta.

### Por qué se siembra por API y NO por la UI
Esta es la pregunta del millón. **Sembramos la precondición por API (llamadas HTTP directas al backend), no haciendo clics en la pantalla.** Razones:
1. **Velocidad:** crear un residuo por la UI son 15 clics y 20 segundos. Por API es **una** llamada HTTP en 200ms.
2. **Robustez:** la UI puede fallar, tener un spinner colgado, un combo lento… y entonces tu test falla por la *preparación*, no por lo que querías probar. La API es directa y estable.
3. **Claridad:** si tu test de "anular" falla, querés saber que falló **anulando**, no creando la SS de apoyo. Separás preparación (API, rápida) de la prueba real (UI, lo que importa).

**La regla:** *"Sembrá por API, verificá por UI."* La preparación va por el atajo rápido; **la comprobación de lo que ve el usuario va siempre por la pantalla** (esa es la regla de oro #2 del repo — un `200` del backend NO prueba que el usuario lo VE bien).

### Cómo funciona en este repo: los setups y el token

Cuando corrés la suite, ANTES de cualquier test corren unos **setups** (definidos en `playwright.config.ts`). El orden lo maneja Playwright con `dependencies`:

```
ambiente  →  auth        →  seed         →  aio (tus tests)
(¿back OK?)  (login admin)  (siembra data)   
             auth-cliente  (login cliente)
```

- **`ambiente`** → chequea que el backend responde. Si está caído, no tiene sentido seguir.
- **`auth`** (`auth.setup.ts`) → hace **UN** login por UI en toda la corrida, y guarda dos cosas: la **sesión** (para reinyectarla en cada test sin re-loguear) y el **token** crudo de la API.
- **`auth-cliente`** (`auth-cliente.setup.ts`) → lo mismo pero para el usuario del **portal cliente** (sesión guardada en `.auth-cliente/`, separada de la del admin):
  ```ts
  setup('Capturar sesion cliente (Zona de Clientes)', async ({ browser }) => {
    const { username, password } = requireCredentialsCliente();
    const auth = new AuthSessionService(authFile);
    await auth.captureSesionLogin(browser, username, password);
  });
  ```
- **`seed`** (`seed.setup.ts`) → siembra la data GENERAL por API, usando el token que capturó `auth` (por eso `seed` **depende de** `auth`, no se re-loguea):
  ```ts
  const api = new ApiRequest();
  api.setToken(auth.readTokenApi());          // reusa el token, no vuelve a loguear
  const data = new DataService(api);          // el "sembrador"
  await data.ensureUbicacion({ departamento: 'ATLANTICO', municipio: 'BARRANQUILLA', ... });
  ```

**El token, en simple:** cuando te logueás, el backend te da un "pase" (una cadena larga de caracteres) que prueba quién sos. En vez de re-loguearte para cada llamada API, guardás ese pase y lo mandás en cada request. `auth` lo captura una vez; `seed` y los tests lo reusan.

### La palabra clave: `ensure` (get-or-create idempotente)
Fijate que el método se llama `ensureUbicacion`, no `crearUbicacion`. La diferencia es enorme:
- **`ensure` = "asegurate de que exista":** primero **busca** si ya existe; si sí, lo reusa; si no, lo crea. A esto se le llama **get-or-create idempotente** ("idempotente" = corras 1 vez o 100, el resultado final es el mismo: existe uno, no 100 duplicados).
- **Por qué importa:** si sembraras con `crear` a secas, cada corrida dejaría basura acumulada (100 ubicaciones "BARRANQUILLA"). Con `ensure`, la primera corrida la crea y las siguientes la encuentran. Limpio y repetible.

### Cada test crea su propia data (semilla puntual dentro del spec)
Además de la semilla general del `seed.setup.ts`, **cada test siembra lo suyo** cuando necesita algo muy específico. Ejemplo real, el caso 04 crea por API un residuo no-aprobado:
```ts
/** Crea por API (token cliente) un residuo del cliente en estado NO aprobado (Solicitado)
 *  → NO debe ofrecerse en el combo Residuo de la SS. */
async function crearResiduoSolicitado(api: ApiRequest, nombre: string): Promise<number> {
  const creado = await api.post<{ data: { id: number } }>('/api/waste-requests/v0/store', {
    company_id: 0, waste_type_id: 5, waste_stream_id: 98, risk_type_id: 21, waste_state_id: 2,
    name: nombre, description: 'no aprobado (caso 04)', controlled_waste: false, active: true,
    request_status: 'Solicitado',
  });
  return creado.data.id;
}
```
Y en el test se usa así, con limpieza al final:
```ts
try {
  idNoAprobado = await crearResiduoSolicitado(api, nombreNoAprobado);   // SIEMBRO por API
  // ... abro el combo por UI y compruebo que NO ofrece ese residuo ...
  expect(residuos, 'el combo NO debería ofrecer un residuo en estado Solicitado').not.toContain(nombreNoAprobado);
} finally {
  if (idNoAprobado) await api.delete(`/api/waste-requests/v0/delete/${idNoAprobado}`).catch(() => {});   // LIMPIO
}
```
- Siembro la precondición por API (rápido y estable).
- Verifico el comportamiento por UI (lo que ve el usuario).
- Limpio en el `finally` (borro el residuo de apoyo pase lo que pase).

> **Regla de oro de la data (memoria del repo):** *nunca mutar registros preexistentes de QA.* Si tu test modifica un cliente que ya estaba, eso es **irreversible**, invalida las aserciones de otros y "gasta" el pool de datos hasta que la suite empieza a fallar por falta de material. **Tu test crea lo que necesita, lo usa, y (cuando aplica) lo limpia.**

### El `ApiRequest`: tu cliente HTTP
`aio/services/api/Apirequest.ts` es el "motor" para hablar con el backend. Lo usás para sembrar y para verificar por atrás:
```ts
const api = new ApiRequest();
api.setToken(authCliente.readTokenApi());   // le doy el pase (token) del cliente
const show = await api.get(`/api/service-requests/v0/show/${id}`);   // GET: leo
await api.post('/api/waste-requests/v0/store', { ...datos });         // POST: creo
await api.delete(`/api/waste-requests/v0/delete/${id}`);             // DELETE: borro
await api.dispose();   // al terminar, cierro el cliente (libera recursos)
```

---

## 7. Cómo correr y leer resultados

### Los comandos (se ejecutan en la terminal, en la raíz del repo)
```bash
npm run test:aio          # corre TODOS los tests de la web AIO
npm run test:ami          # corre los de móvil (necesita device + appium)
npm run test:e2e          # corre los cross-app
npm run test:api          # smoke de la capa API
npm run test:ambiente     # solo chequea que el backend responde
```
- `npm run` ejecuta un "atajo" definido en `package.json`. `test:aio` por dentro llama a Playwright con la configuración correcta.

### Correr UN solo test (clave mientras desarrollás)
No re-corras 18 tests cuando estás iterando sobre uno. Filtrá por el nombre con `-g` ("grep"):
```bash
npm run test:aio -- -g "01-Crear una SS con entrega Planta"
```
Por eso los casos se numeran (`01-`, `16-`): el número hace de "handle" para agarrar uno.

**Correr solo un archivo (spec) y sin re-hacer los setups pesados**, cuando ya tenés sesión válida:
```bash
npx playwright test aio/tests/modules/zona-clientes/ClientesSolicitudServicio.spec.ts --project=aio --no-deps
```
- `--project=aio` → corré con la config del proyecto `aio`.
- `--no-deps` → NO re-ejecutes los setups (`auth`/`seed`) — útil cuando ya corrieron y solo querés re-probar el spec. (Ojo: la primera vez del día SÍ necesitás los setups, así que no uses `--no-deps` hasta tener sesión.)

### Cómo leer un resultado

**Verde (pasó):** el caso hizo todo y todas las aserciones se cumplieron. Listo.

**Rojo (falló):** Playwright te dice **en qué línea** falló y **qué esperaba vs. qué obtuvo**. Por eso escribimos el mensaje en cada `expect`:
```ts
expect(show.data.status_service, 'la SS debería nacer en "Solicitada"').toBe('Solicitada');
```
Si falla, ves: *"la SS debería nacer en Solicitada — Expected: 'Solicitada', Received: 'Procesada'"*. Sin adivinar.

**Dónde mirar cuando falla:**
1. **El mensaje del `expect`** en la terminal → qué aserción se rompió.
2. **El screenshot `test-failed-1.png`** → Playwright saca una foto automática al fallar (`screenshot: 'only-on-failure'` en la config). Está en `test-results/.../`. Te muestra **la pantalla en el momento del error**.
3. **El video** → también se guarda solo en fallos (`retain-on-failure`). Ver el video muestra la película completa.
4. **`error-context.md`** → un volcado del estado de la página al fallar.
5. **`test-results/.last-run.json`** → qué tests fallaron en la última corrida (para triage).

### La regla de oro al triagear un rojo
> **Cuando un test falla, tu PRIMERA acción NO es editar el código. Es explorar EN VIVO el paso que falló.**

El error de consola se captura **después** del fallo y puede ser impreciso (la causa real suele estar *antes*). Abrí la pantalla de verdad, repetí el paso a mano, y mirá qué pasa realmente. Recién con esa evidencia arreglás. **Editar "de oído" es la trampa #1** y hace perder corridas enteras. (Hay hasta un hook en el repo que te bloquea la 3ª edición a ciegas del mismo archivo, para forzarte a explorar.)

---

## 8. Errores típicos + reglas de oro

### Errores típicos de principiante (y cómo evitarlos)
1. **Escribir el test sin haber mirado la pantalla.** → Explorá primero. Inventar un selector = bug garantizado + corrida perdida.
2. **Usar `waitForTimeout(3000)` para "que cargue".** → Esperá por efecto (`waitForResponse`, `toBeVisible`), nunca por tiempo fijo.
3. **Selectores por índice o por id autogenerado.** → Por `name`/rol/texto. Índice y `#...-sgov2` se rompen.
4. **Reusar un registro que ya estaba en QA.** → Cada test crea su propia data (con `Date.now()` para que sea única).
5. **Modificar el POM ajeno para "que me sirva".** → Composición: reusá sin tocar. Si de verdad hay que cambiarlo, preguntá.
6. **Dar por bueno un `200` del backend.** → Un 200 no prueba que el usuario lo VE. Verificá por la UI.
7. **No probar los negativos.** → El camino feliz esconde pocos bugs. Los negativos (campo readonly, dato inválido, acción prohibida) esconden muchos.
8. **Editar el código apenas ve un rojo, sin explorar.** → Primero evidencia en vivo, después el fix.

### Las reglas de oro del repo (memorizalas — son la cultura de acá)
- **🥇 NO ASUMIR NADA.** Si no lo verificaste con evidencia, **no lo sabés**. Los bugs se esconden en los detalles "obvios": un "Ver" que resulta editable, un dato que se envía pero no se muestra, un combo que se ve lleno pero trae el valor equivocado. Ante cualquier *"seguro funciona"* → **PARÁ y comprobalo.**
- **🚦 Explorar antes de codear.** UI que no viste con tus ojos → explorala en vivo ANTES de escribir el método. Test que falla → explorá el paso que falló en vivo, no edites de oído.
- **🎯 Somos un usuario más: todo por la UI.** Verificá por la pantalla (no por atajos), y **ejercé las acciones negativas**: en un "Ver"/solo-lectura, **intentá editar y guardar** para confirmar que bloquea. Un "Ver" que deja editar es un bug GRAVE — se nos pasó una vez por no intentarlo. (Y cuando pruebes en vivo, interactuá como persona real: clics y tecleo, nunca manipular por JavaScript — eso falsea la prueba.)
- **No tocar lo que ya funciona.** Agregá por composición; atacá la causa raíz, no el síntoma.
- **Mantené el cerebro actualizado.** Cada comportamiento real que descubrís va al `docs/context/...` del módulo. El cerebro es "la biblia" del repo.

---

## 9. Mini-ejercicio final

Ahora te toca a vos. **No mires la solución de los casos existentes hasta intentarlo.** El objetivo es que pases por todo el flujo solo/a.

### El reto
> Escribí un test nuevo (llamalo `19-...`) en el mismo spec que valide esto:
> **"Al crear una SS con recolección Ruta y frecuencia Diaria, si elijo MENOS de 2 días la SS NO se debe poder guardar"** (regla de negocio: Diaria exige ≥2 días).

### Pistas (sin darte la solución)
1. **Primero el doc:** releé en el cerebro (`docs/context/aio/zona-de-clientes/respel/solicitud-de-servicios.md`) la parte de *Frecuencia/Semana-Día* y los gotchas. ¿Qué dice sobre "Diaria exige las 4 semanas + ≥2 días"?
2. **Explorá en vivo:** entrá al portal cliente, empezá a crear una SS Ruta, andá a la pestaña "Frecuencia del Servicio" y probá a mano elegir **1 solo día**. ¿Qué hace el sistema? ¿Sale un mensaje? ¿Cuál es el texto exacto? Apuntalo (ese texto va a ser tu aserción).
3. **Reusá el POM, no reinventes:** mirá cómo el caso 02 usa `ss.crearRuta(...)` y cómo se le pasan `{ frecuencia: 'Diaria', semanas: [...], dias: [...] }`. Para tu caso vas a querer pasar un solo día (`dias: ['Lunes']`) y esperar que **NO** guarde.
4. **Estructura:** copiá el esqueleto de un caso existente (el `test('19-...', async ({ page }, testInfo) => {`), poné tu descripción con `attachLoginScenarioData`, hacé la acción y la aserción.
5. **La aserción clave:** parecida al caso 05 (`intentarGuardarSinResiduo`) — comprobá que `guardo === false` y que el toast contiene el mensaje que viste en el paso 2. Recordá: **probás que NO se guardó** (un negativo).
6. **Corré SOLO tu caso:** `npm run test:aio -- -g "19-"`.
7. **Si sale rojo:** no edites de una. Volvé al paso 2, mirá en vivo qué pasó de verdad, y recién ahí ajustá.

### Cómo sabés que lo hiciste bien
- Tu test es **verde** y, si comentás la regla (elegís 2 días), **cambia** de comportamiento (o sea, tu test de verdad discrimina, no es un falso verde).
- No tocaste ningún POM ajeno.
- La data que creaste es única (`Date.now()`) y no dependés de nada preexistente.
- Cuando termines, si descubriste algo nuevo del comportamiento real, **anotalo en el cerebro**.

Cuando lo tengas, comparalo con el caso 05 y el 02 para ver cómo lo resolvió otra persona. Vas a notar que llegaste a algo muy parecido — y eso significa que **ya sabés automatizar**. 🎉

---

## Chuleta rápida (para pegar en tu pared)
| Necesito… | Uso… |
|---|---|
| Buscar un elemento estable | `getByRole` / `getByLabel` / `[name="..."]` — nunca índice ni id autogenerado |
| Plan B de selector | `.locatorA.or(locatorB).first()` |
| Esperar que cargue | `waitForResponse(...)` o `expect(x).toBeVisible()` — nunca `waitForTimeout` |
| Comprobar algo | `expect(valor, 'mensaje claro').toBe(...)` |
| Comprobar que algo NO está | `expect(x).toHaveCount(0)` |
| Preparar precondición | sembrar por **API** (`ApiRequest` / `DataService.ensure...`) |
| Verificar lo que ve el usuario | por **UI**, siempre |
| Data única por test | `` `Texto QA ${Date.now()}` `` |
| Reusar un POM sin tocarlo | **composición**: `private ss = new OtroPage(this.page)` |
| Correr un solo caso | `npm run test:aio -- -g "NN-titulo"` |
| Entender un módulo rápido | leer su **cerebro** en `docs/context/...` |

> **La regla que resume todo:** *si no lo verifiqué con evidencia, no lo sé.* Automatizar es convertir esa desconfianza sana en código que comprueba, una y otra vez, que el producto se comporta como debe.
</content>
</invoke>
