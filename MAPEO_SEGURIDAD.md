# MAPEO DE SEGURIDAD — CRM Contable (demo) y SueldoNET (demo)

> Los dos son **demos de un solo archivo HTML, sin servidor ni cuentas**: todo vive en el `localStorage` del navegador
> de quien los abre y los datos son de ejemplo. Por eso casi no hay superficie de ataque hoy. Este mapeo existe para
> dejar escrito **qué cambia el día que se usen con datos reales**.

**Última revisión:** 4/10/2026 — lectura de código y búsqueda de secretos, llamadas de red y uso de `innerHTML`. Sin pruebas automáticas
(no hay servidor que probar). SueldoNET es `Apps para Contador/mockup_talon.html` (fuera de este repo).

## Qué se revisó

| Área | Resultado |
|---|---|
| Secretos / claves | no hay claves de API ni tokens en `app/index.html`, `index.html` ni `mockup_talon.html` ✅ |
| Llamadas de red | CRM Contable hace solo 2: `GET /api/cotizacion-bcu` y `POST /api/sugerencia-externa` al CRM Notarial (públicos a propósito; el segundo con tope diario por IP). SueldoNET: ninguna ✅ |
| XSS | `app/index.html` usa `innerHTML` 37 veces y un helper `esc()` (51 usos); como cada usuario solo ve los datos que cargó él mismo en su navegador, el riesgo es de auto-XSS, no de ataque a terceros ✅ (bajo) |
| `eval` / `new Function` | ninguno ✅ |
| Librerías externas | `echarts` desde cdnjs, sin atributo `integrity` (SRI) ⚠️ |
| Autenticación | **simulada en el navegador.** El "portal del cliente" (`codigoAcceso`) y el login de SueldoNET (CI + empresa + contraseña `1234` en el propio código) no protegen nada: cualquiera puede leer los datos desde las herramientas del navegador o el código fuente |

## Regla para el día que esto salga de demo
1. **No cargar datos reales de clientes ni de empleados** (CI, sueldos, recibos) en estas versiones. Es información personal y sensible (Ley 18.331).
2. Pasar a **backend real con cuentas por cliente**, como el CRM Notarial (su `MAPEO_SEGURIDAD.md` es el modelo: sesión atada a la contraseña, límites de login, aislamiento por cuenta, PIN/enlace con freno de intentos).
3. En SueldoNET, además: guardar **hash** de contraseñas (nunca `'1234'` en el código), y proteger la descarga del PDF original del recibo con la misma sesión.
4. Ponerle `integrity` + `crossorigin` a las librerías de CDN, o servirlas desde el propio sitio.

## Falta
- Nada que corregir hoy. Re-mirar al convertirlas en producto con backend.
