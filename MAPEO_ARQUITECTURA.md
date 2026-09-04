# CRM Contable ("Libro Mayor") — mapeo de arquitectura

Ruta: `Apps para Contador/crm-contable-demo/`
Repo git propio (remoto: `github.com/Nandoezequiel/crm-contable-demo`), commit inicial `4dab605` (31/8/2026), último commit `a000119`.

**Estado real, verificado leyendo el código (3/9/2026): es 100% frontend.** No hay ni un `fetch`, `XMLHttpRequest` ni referencia a un backend en todo el proyecto — cero llamadas de red de negocio. Todo el estado vive en `localStorage` del navegador. A diferencia del CRM Notarial (Flask + SQLAlchemy + SQLite, multi-tenant, desplegado en Render), esto sigue siendo un mockup de un solo archivo, sin servidor propio.

## 1. Los dos archivos HTML: no son versiones, son piezas distintas

`index.html` (raíz, 745 líneas) y `app/index.html` (2136 líneas) **nacieron juntos en el mismo commit** (`4dab605`, "Sitio de presentación del CRM Contable, con el CRM en /app") — no hay un `index.html` viejo reemplazado por `app/index.html`; son dos productos con roles distintos, coexistiendo a propósito:

- **`index.html`** = landing/pitch comercial. Un slide-deck de 10 diapositivas navegables (flechas, puntos, teclado, swipe táctil) con estética "libro mayor" (folios, líneas de cuaderno). Presenta el problema, la solución, beneficios puertas adentro/hacia afuera, un panel ilustrativo con datos inventados, y anticipa los otros dos productos de la suite (Minutario Contable, SueldoNET). Su único uso de `sessionStorage` es recordar en qué diapositiva quedó el visitante (`crmcontable_slide`) — no toca ningún dato de negocio. Termina con un botón que linkea a `app/` (el CRM real).
- **`app/index.html`** = el CRM en sí, la app funcional. Es el archivo vigente para cualquier trabajo de producto; `index.html` es solo la puerta de entrada comercial y no requiere tocarse salvo que cambie el mensaje de venta.

No hay ambigüedad de "cuál es el vigente": ambos lo están, cada uno para su propósito. Si alguien pide "arreglar el CRM", es `app/index.html`. Si pide "arreglar la presentación/landing", es `index.html`.

## 2. Persistencia: localStorage, con convención de versión en la clave

Confirmado en `app/index.html:893-901` — sigue la misma convención de bump de versión (`_V1`, `_V3`, `_V4`) que otros proyectos del usuario ([[feedback-localstorage-seed-version-bump]]):

```js
const LS = {
  clientes:     'LM_CLIENTES_V4',
  obligaciones: 'LM_OBLIG_CATALOGO_V3',
  clienteOblig: 'LM_CLIENTE_OBLIG_V3',
  cobros:       'LM_COBROS_V4',
  tareas:       'LM_TAREAS_V1',
  tc:           'LM_TC_V1',
  prospectos:   'LM_PROSPECTOS_V1'
};
```

El prefijo `LM_` viene de "Libro Mayor" (nombre de trabajo del producto, visible en el `<title>` de `index.html`). Cada clave se siembra con datos de ejemplo solo si `localStorage.getItem(key)` viene vacío (`load()` con fallback `null` + `if(!X){ ... seed ...}`), el mismo patrón `load`/`save` ya usado en otros mockups del usuario. Si se vuelve a tocar el seed de cualquiera de estas colecciones, corresponde subir su número de versión — igual que ya se hizo varias veces en CRM Notarial (`LS_COBROS` v5→v6, `LS_TAREAS` v1→v2) antes de que ese proyecto tuviera backend real.

Aparte del tema visual (`UI_TEMA`, claro/oscuro, compartido con la landing vía `data-theme`), no hay ninguna otra persistencia — no hay IndexedDB, no hay cookies, no hay backend que respalde nada.

## 3. Qué está mockeado con datos de ejemplo vs. qué es funcionalidad real

Esta es la distinción más importante del documento: **casi todo lo que parece "una acción" en esta app realmente ejecuta lógica JS real y persiste en localStorage** — no son solo botones decorativos. Lo que es mock es el *contenido inicial* (11 clientes, 4 prospectos, ~6 meses de cobros históricos, 13 tipos de obligación DGI/BPS), no el motor.

### Funcional de verdad (persiste en localStorage, con lógica real)
- **Clientes**: alta/edición completa (régimen fiscal, honorario, moneda, prioridad, carpeta Drive/OneDrive, código de acceso al portal), búsqueda, orden por columna.
- **Obligaciones (catálogo)**: 13 tipos precargados (IVA, IRPF, IRAE, BPS, certificados, etc.) con periodicidad en días; se pueden asignar/desasignar por cliente con fecha de último cumplimiento, y el sistema calcula el estado (vencido/próximo/al día) a partir de esa fecha + periodicidad.
- **Cobros**: honorarios recurrentes + cobros puntuales por etiqueta libre, con pagos parciales (`pagos: []`) y estado derivado (pendiente/parcial/cobrado). Botón "Facturar mis honorarios del mes" genera un cobro individual por cada cliente activo sin duplicar el mes ya facturado (con vista previa antes de confirmar).
- **Prospectos**: pipeline real con 5 etapas (Contactado → Reunión → Propuesta → Ganado/Perdido), filtrable.
- **Agenda**: calendario mensual navegable con vencimientos + tareas propias por día; permite crear tareas ligadas a un cliente y generar un link real a Google Calendar (`calendarLink()`, sin OAuth — mismo patrón "wa.me" de link directo sin backend ya usado en CRM Notarial).
- **Portal del cliente** (`#portal`, ruta separada por hash): cada cliente tiene un `codigoAcceso` (ej. `DELTA-101`) que, si coincide, muestra un dashboard de solo lectura con sus propias obligaciones, saldo pendiente e historial de cobros. **Esto ya es más de lo que tiene hoy el CRM Notarial** (que solo tiene esta idea anotada como proyecto futuro sin construir — ver sección 4).
- **WhatsApp**: link `wa.me/<tel>?text=` con mensaje contextual precargado por cliente (mismo patrón sin Business API que ya se adoptó en CRM Notarial).
- **Exportar "Excel"**: genera y descarga un **CSV real** (`Blob` + `URL.createObjectURL`), no un `.xlsx` — funciona de verdad (clientes y cobros filtrados/ordenados tal como se ven en pantalla), pero el rótulo del botón ("⇩ Exportar Excel") es más ambicioso que el archivo que entrega.
- **Estado de cuenta imprimible**: arma un HTML aparte (`#imprimible`) con las obligaciones y el historial de cobros del cliente y dispara `window.print()` — funciona en cualquier navegador vía CSS `@media print`, sin backend.
- **Modo claro/oscuro**: persistente, compartido con la landing.

### Solo visual / con datos fijos que no llegan a ser un flujo editable
- El **panel ilustrativo de `index.html`** (la landing): cifras de ejemplo puestas a mano en el HTML, marcadas explícitamente como `<p class="illustrative">Panel ilustrativo — cifras de ejemplo, no datos reales de un estudio.</p>`. No lee ningún dato real del CRM.
- Los gráficos del Panel real (`app/index.html`, vía ECharts 5.6 por CDN) sí son dinámicos — se recalculan de los datos en localStorage — pero como todo el proyecto no tiene multiusuario ni fuente de verdad compartida, "dinámico" acá solo significa "reactivo a lo que hay en el navegador de quien lo abre", no a datos reales de un estudio.
- Cotización de referencia USD→UYU: el usuario la carga y edita a mano (`TC.valor`); no se descarga de ninguna API — el propio tooltip del widget lo aclara.

## 4. Comparación con el CRM Notarial: qué patrones ya validados convendría reutilizar

El CRM Notarial (`Apps para Escribana/CRM_SISTEMA/`) ya resolvió, en producción real, varias decisiones que este proyecto va a enfrentar el día que deje de ser un mockup. Vale la pena copiar el patrón, no reinventarlo:

| Decisión | CRM Notarial (ya resuelto) | Aplicaría igual al CRM Contable |
|---|---|---|
| **Stack backend** | Flask + SQLAlchemy + SQLite | Mismo stack — ya validado en dos proyectos del usuario (Notarial y, con JSON en vez de SQL, MOTORMAX) |
| **Multi-tenant** | Una fila `Escribana` (o `Contador` acá) por cuenta; todo (`Cliente`, `Cobro`, `Tarea`) lleva su FK — sin login por asiento ni "estudio" separado de "usuario" hasta que haga falta de verdad | Igual: una tabla `Contador` y todo el resto colgando de `contador_id`. No construir roles/equipo hasta que un estudio con varios contadores lo pida en serio |
| **Autenticación** | Cuenta + contraseña por escribana, registro público cerrado por variable de entorno, alta manual vía script (`crear_escribana.py`) | El **PIN por cliente que ya existe en el portal de este demo NO alcanza para el contador mismo** — el contador necesita login real con contraseña, el PIN queda reservado para sus clientes finales (que es justo el modelo que el Notarial todavía tiene pendiente de construir, ver abajo) |
| **Migraciones de esquema** | Sin Alembic — scripts `ALTER TABLE` idempotentes a mano (`migrar_finanzas.py`, `migrar_cliente_columnas.py`), corridos manualmente en el Shell de Render cada vez que se agrega una columna a una tabla que ya tiene datos reales | Mismo patrón: nunca confiar en `db.create_all()` para una tabla ya poblada |
| **Términos y condicionales / consentimiento de datos** | Gate real de T&C (Ley 18.331, alojamiento fuera de Uruguay) aceptado una vez por cuenta, guardado en la fila del usuario | Igual — el CRM Contable también guarda datos personales de clientes de terceros (CI/RUT, teléfono, honorarios) |
| **Hardening HTTP** | Headers de seguridad (`X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Strict-Transport-Security`) vía `@app.after_request` | Copiar el mismo hook tal cual |
| **Despliegue** | Render Starter pago + disco persistente 1GB, nunca nivel gratis para datos reales; `Procfile` + `requirements.txt` en la raíz | Mismo criterio — el usuario ya decidió no usar el free tier para datos de clientes reales en ningún proyecto |
| **Exportar a Excel real** | `openpyxl` con estilos (encabezado con color de marca, freeze panes, columna de estado coloreada) | El CSV actual de este demo se reemplazaría por el mismo enfoque `openpyxl`, ya escrito y probado en el otro proyecto |

**Dato curioso en sentido inverso:** el "estado de cuenta imprimible" de este mismo demo (`imprimirEstadoCuenta()`) fue la referencia explícita que el usuario dio para la idea de "portal para los clientes del escribano" que quedó anotada como proyecto futuro *sin construir* en el CRM Notarial. Es decir, en el vertical notarial el CRM Contable es la inspiración; y en el propio CRM Contable, el portal con PIN individual (`codigoAcceso`) **ya está construido** — es un caso donde este demo va un paso adelante del hermano notarial en una feature concreta, y ese código (login por código, dashboard de solo lectura) es reutilizable casi tal cual si el Notarial retoma esa idea.

## 5. Qué haría falta para pasar de mockup a producción real

Siguiendo el mismo criterio que se usó para decidir cuándo el CRM Notarial pasó de Artifact a Render:

1. **Backend real**: Flask + SQLAlchemy + SQLite (o Postgres si el volumen lo justifica), con modelos `Contador`, `Cliente`, `ObligacionCatalogo`, `ClienteObligacion`, `Cobro`, `Pago`, `Tarea`, `Prospecto` — mapeo casi directo de los arrays JS actuales (`CLIENTES`, `OBLIGACIONES`, `CLIENTE_OBLIG`, `COBROS`, `TAREAS`, `PROSPECTOS`).
2. **Autenticación real** para el contador (login + contraseña, registro cerrado, alta manual) — hoy no hay ninguna, cualquiera que abra el archivo ve y edita todo.
3. **Reescribir `app/index.html`** para hablar con una API (`fetch`) en vez de leer/escribir `localStorage` directamente — mismo trabajo que ya se hizo con `frontend.html` en el CRM Notarial.
4. **Migrar el portal de código de cliente** (`codigoAcceso`) al modelo real: hoy el código vive en texto plano en el array de clientes; en producción necesita, como mínimo, no ser adivinable en bloque (son 11 códigos consecutivos previsibles como `DELTA-101`, `NUNEZ-102`...) y estar detrás de un endpoint separado del login del contador.
5. **Gate de Términos y Condiciones** (Ley 18.331, igual que el Notarial) antes de guardar datos reales de clientes de terceros.
6. **Hardening HTTP** (los mismos 4 headers) y despliegue en Render Starter pago + disco persistente — nunca free tier con datos reales, mismo criterio ya aplicado dos veces.
7. **Exportar a Excel de verdad** con `openpyxl` en vez del CSV actual, reutilizando el código ya escrito para el Notarial.
8. **Decidir qué pasa con `index.html`** (la landing): probablemente se mantiene como está, sirviendo estático, y solo el link final ("Abrir el CRM Contable en vivo") pasa de `app/` a la URL real de producción.

Ninguno de estos pasos está empezado a día de hoy (3/9/2026) — el proyecto sigue siendo, íntegramente, el mockup descrito en las secciones 1-3.

## 5.1 Brecha frente al CRM Notarial — cerrada en parte (3/9/2026)

Tras mapear también `Apps para Escribana/CRM_SISTEMA/MAPEO_ARQUITECTURA.md` (mucho más avanzado: backend Flask real, multi-tenant, 2FA, panel de admin), se comparó feature por feature y se decidió portar al mockup — sin backend, todo JS + localStorage — las mejoras que no dependen de tener servidor. Implementado y verificado end-to-end con Playwright (0 errores de consola):

1. **Cerrar/reabrir cliente** — botón en la ficha (`btn-cerrar-reabrir-cliente`), toggle "Ver clientes cerrados" en el toolbar de Clientes. El campo `activo` ya existía en el dato pero nada lo cambiaba; ahora sí.
2. **Regenerar código de acceso del portal** — botón junto al código en la ficha, con confirmación previa (invalida el anterior de inmediato).
3. **Selección múltiple / acciones en lote en Clientes** — botón "✏️ Seleccionar", checkboxes por fila, barra para cambiar prioridad o cerrar varios a la vez.
4. **Vista "Cobros por persona"** — toggle "Por trámite" / "Por persona" en Cobros; la vista por persona agrupa y expande inline (mismo patrón `.collapse-body` que ya usaba la ficha de cliente).
5. **Cronograma de cuotas** — campo opcional "Cuotas" al cargar un cobro puntual (`generarCuotas()` arma el plan de N cuotas mensuales iguales); `progresoCuotas()`/`chipCuotas()` muestran "Cuota X de N" en la tabla de Cobros, en el historial del cliente y en la vista por persona.
6. **Etiquetas de Cliente** (además de las etiquetas de Cobro que ya existían) — chips + input con autocompletado en la ficha; el buscador de Clientes ya matchea por etiqueta.
7. **Notificaciones de escritorio** (Chrome Notification API) — widget en el sidebar, minutos de anticipación configurables (1-180, default 15) guardados en `localStorage` (no en el dato de negocio, mismo criterio que `UI_TEMA`). Avisa por tareas con hora X minutos antes, y un resumen una vez por día si hay vencimientos vencidos o que vencen hoy.
8. **Tareas: campo Hora + apertura automática de Google Calendar** — antes ese puente solo existía para vencimientos DGI/BPS; `calendarLink()` se extendió para aceptar hora (arma un evento puntual en vez de todo el día).
9. **WhatsApp — mensaje de "estado de cuenta"** (`mensajeEstadoCuenta()`) — resumen no itemizado con el link del portal (`#portal`) y el código de acceso incluidos, distinto del mensaje contextual existente (que sigue igual, por vencimiento o cobro puntual).
10. **Buscador de Clientes ampliado** — ahora matchea también teléfono, notas y etiquetas (antes solo nombre y CI/RUT).

**Bump de versión de localStorage** (mismo criterio que en otros proyectos, [[feedback-localstorage-seed-version-bump]]): `LM_CLIENTES_V4→V5` (etiquetas de ejemplo agregadas a varios clientes), `LM_COBROS_V4→V5` (cobro de ejemplo con plan de cuotas para Carpintería Bianchi, 1 de 3 pagada), `LM_TAREAS_V1→V2` (hora agregada a una tarea de ejemplo).

**No portado** (queda en la lista de brecha, requiere backend real primero): exportar a `.xlsx` con `openpyxl`, autenticación real del contador, panel de administración con impersonación, buzón de soporte, gate de Términos y Condiciones por cuenta, migraciones de esquema vía script — sin cambios, siguen documentados en la sección 5 de arriba.

## 5.2 Segunda tanda (4/9/2026) — flujo de Cobros, cotización real, Finanzas personales

Tras una segunda revisión exhaustiva del CRM Notarial (pestaña por pestaña, incluyendo "Mi cuenta") se implementó todo lo que no depende de backend, en un solo lote grande. Verificado end-to-end con Playwright (28 pasos, 0 errores de consola).

**De flujo (los 8 que quedaban pendientes de la primera comparación):**
1. **Formulario "Nuevo cobro" directo en la pestaña Cobros** — combo de cliente por texto libre (`<input list>` con datalist) + validación (si el texto no matchea un cliente real, error inline + borde rojo, no deja guardar). Antes había que abrir la ficha del cliente para cargar cualquier honorario puntual.
2. **Editar y eliminar un cobro ya creado** — modal nuevo `#modal-editar-cobro` (concepto/monto/moneda/fecha + botón eliminar con confirm), accesible desde las 3 vistas donde aparece un cobro (tabla, vista por persona, historial de cliente).
3. **Editor completo de cuotas** en ese mismo modal — generar un plan de N cuotas mensuales iguales (`generarCuotas()`), editar fecha/monto de cada una a mano, guardar o quitar el plan completo. Antes las cuotas solo se definían una vez, al crear el cobro.
4. **Modo selección en Cobros** (mismo patrón que ya existía en Clientes) — marcar varios pendientes como cobrados o eliminarlos en lote, respetando el filtro/búsqueda activo para "seleccionar todos".
5. **Botón "Ver ficha completa del cliente →"** desde la vista por persona de Cobros — abre directo el modal de ficha en Clientes, sin tener que ir a buscarlo.
6. **Buscador de Clientes cruza contra conceptos de cobros históricos** — buscar "auditoría" encuentra al cliente que tuvo ese trabajo, no solo por sus datos propios.
7. **Botón "Hoy" en la navegación de Agenda.**
8. **Tile de "Tareas pendientes" del Panel desglosado** en Vencidas / Hoy / Próx. 7d (antes un solo número).

**Más 10 mejoras chicas** (normalizar búsqueda ignorando tildes vía `normalizarTexto()`, botón "Copiar link" del portal con clipboard, contador en "Ver clientes cerrados (N)", botón "+Tarea" directo desde la fila de cliente, confirmación al borrar una tarea, botón eliminar prospecto, agregar etiqueta en lote de Clientes, timezone `America/Montevideo` + `<a>` clickeado en vez de `window.open()` en los links de Calendar, sistema de toast (`mostrarToast()`) para las acciones en lote, botón "Cuota N cobrada" que precarga el monto exacto de la próxima cuota).

**Cotización de USD automática (BCU vía terceros):** investigado el estado real del BCU — su único webservice es SOAP (`cotizaciones.bcu.gub.uy`), no apto para llamar directo desde un frontend estático sin backend (sin CORS confirmado). Se encontró y usó en su lugar `https://uy.dolarapi.com/v1/cotizaciones/usd` — confirmado con `curl` que responde JSON con `Access-Control-Allow-Origin: *`, y es el mismo endpoint que el usuario ya usaba en su calculadora de financiamiento de VIDESOL (`videsol-presentaciones-main/videsol-calculadora-financiamientoV2.html`), con el mismo criterio de tomar `venta` (o `compra`/`valor` si falta) y fallback silencioso al valor manual si el fetch falla. Botón "🔄" nuevo junto al TC del sidebar; además se intenta actualizar solo una vez por día al abrir la app (si `TC.fecha !== hoy`), sin bloquear nunca la carga si no hay red. El campo `TC.fuente` (`'manual'`|`'auto'`) queda visible en el tooltip de fecha para que quede claro de dónde salió el valor. Es una fuente de terceros no oficial — para producción real, lo ideal sigue siendo que un backend futuro consulte el SOAP oficial del BCU y lo cachee.

**Módulo de Finanzas personales completo** (existía en el CRM Notarial, no en Contable) — portado entero como pestaña opt-in nueva (oculta por defecto, activable con un checkbox en el sidebar, mismo criterio que "Notificaciones"): 4 sub-pestañas (Resumen, Gastos fijos, Gastos variables, Historial), con localStorage propio (`LM_FIN_FIJOS_V1`, `LM_FIN_VARIABLES_V1`, `LM_FIN_HISTORIAL_V1`, `LM_FIN_CONFIG_V1`). Al activarlo por primera vez se siembran 4 gastos fijos de EJEMPLO (Alquiler/Internet/Gimnasio/Seguro, badge "Ejemplo" visible). Resumen con: alerta de saldo negativo/bajo, 5 stat-tiles (ingreso editable inline, fijos, variables, disponible con color dinámico, meta de ahorro editable), lista de próximos vencimientos con semáforo, y 3 gráficos ECharts (donut de distribución, barras de ahorro por mes, barras de evolución de gastos — ambas con el mes en curso marcado con borde punteado, mismo lenguaje visual que el resto del dashboard). "Cerrar mes" archiva un snapshot en el historial y desmarca todos los fijos como no pagados para el mes nuevo. Es un módulo enteramente personal del contador — no toca clientes ni cobros del estudio.

## 5.3 Tercera tanda (4/9/2026) — dos ajustes puntuales

1. **Los modales siempre abren scrolleados arriba.** Bug: `.modal-overlay` es el elemento que scrollea (`overflow-y:auto`), y como queda en el DOM con `display:none` al cerrarse, conservaba el `scrollTop` de la vez anterior — si desplazabas la ficha de un cliente hacia abajo y la cerrabas, la próxima ficha que abrías (la de ese cliente o cualquier otro) aparecía ya desplazada. Fix de una línea en la función genérica `abrirModal(id)`: resetea `scrollTop = 0` cada vez que se abre, así que aplica a todos los modales de la app (cliente, cobro, pago, etc.), no solo al de clientes.
2. **Conversión automática al cambiar la moneda de un cobro ya cargado.** En `#modal-editar-cobro`, cambiar el selector de moneda (UYU↔USD) ahora recalcula el importe con la cotización vigente (`TC.valor`, la misma que alimenta el widget del sidebar) en vez de solo cambiar el símbolo dejando el número tal cual. Si el cobro ya tiene cuotas cargadas, también se convierten proporcionalmente. Si el cobro ya tiene pagos registrados, aparece una confirmación explícita antes de cambiar la moneda ("no se van a convertir, quedan tal cual quedaron cobrados") — los pagos históricos nunca se tocan, porque ya fue plata efectivamente cobrada en esa moneda en su momento; solo se avisa y se deja decidir. Cancelar la confirmación revierte el selector a la moneda original sin tocar nada.

Verificado con Playwright (3 casos: reapertura de modal sin scroll heredado, conversión sin pagos, conversión con pagos incluyendo el camino de cancelar la confirmación) — 0 errores de consola.

## 5.4 Cuarta tanda (4/9/2026) — fix visual del widget de cotización

**El "$" del widget de cotización (sidebar) quedaba en su propia línea, separado del número** (ej. "$" arriba y "41,35" abajo) cuando el widget quedaba con poco espacio horizontal — el navegador partía la línea justo en el espacio entre el signo y el valor, dentro del mismo `<span>` (`#tc-valor-txt`). Fix: `white-space:nowrap` en `.tc-display` (se hereda al span de adentro), así "$ 41,35" queda siempre pegado en una sola línea sin importar el ancho disponible.

## 7. Pendientes anotados por el usuario (para la próxima tanda de cambios)

Estos cambios fueron pedidos explícitamente pero **todavía no se implementaron** — el usuario prefiere juntar varios pedidos y aplicarlos todos de una vez:

1. **Portal de clientes — mostrar el valor de cada obligación.** Hoy la lista "Tus obligaciones" del portal (`#portal`) muestra nombre + organismo + estado (vencido/próximo), pero no el importe de cada una. Pedido (4/9/2026): que se pueda hacer click en cada obligación para ver su valor, o alternativamente agregar una columna con el monto directamente en la lista. Depende de que cada obligación asignada a un cliente (`CLIENTE_OBLIG`) tenga o pueda calcular un importe asociado — hoy el catálogo de obligaciones (`OBLIGACIONES`) no necesariamente tiene un monto por tipo, así que además de la UI hay que decidir de dónde sale ese valor (¿fijo por tipo de obligación? ¿cargado por cliente? ¿el honorario del cliente prorrateado?) antes de implementarlo.

## 6. Relación con MINUTARIO_CONTABLE_SISTEMA (carpeta hermana)

`Apps para Contador/MINUTARIO_CONTABLE_SISTEMA/` es un producto **distinto y ya con backend real** (Flask + librería `anthropic`, según su `requirements.txt`) — es el "Minutario Contable" que la landing de este mismo proyecto anuncia como "en desarrollo" (Fs. 07 de la presentación), con lectura automática de comprobantes por IA. No comparte código, base de datos ni sesión con el CRM Contable demo; son dos apps separadas de la misma suite ("Libro Mayor" vertical contador), igual que SueldoNET es una tercera pieza independiente. Este documento no mapea ese proyecto en detalle — queda para un mapeo propio si se retoma.
