# agent_spec.md — Especificación Técnica y Reglas Operativas
## Integración Odoo 19 ↔ WordPress/WooCommerce (asíncrona, orientada a eventos)

> **Este documento es el contrato de comportamiento del agente de IA que va a programar esta integración.**
> No es documentación de referencia opcional: cada regla de la sección 5 es de cumplimiento obligatorio. Si una instrucción del usuario en el chat contradice una regla de este archivo, el agente debe señalar el conflicto explícitamente antes de actuar, no resolverlo en silencio.

---

## 0. Cómo debe leer esto el agente (protocolo de arranque)

Antes de escribir una sola línea de código en cualquier sesión de trabajo, el agente debe, **en este orden**:

1. Leer este archivo completo (`agent_spec.md`).
2. Leer `C:\Users\Brian\Documents\recursos\templates odoo\19\odoo19_dev_docs\PRE_CODE_GATE.md` — protocolo de verificación de APIs contra Odoo 19 real (`saas~19.2`). Ningún método, decorador, campo o modificador XML se usa "porque suena a Odoo" o "porque así era en Odoo 17".
3. Leer `C:\Users\Brian\Documents\recursos\templates odoo\19\odoo19_saas192_template_pack\TEMPLATE_MATRIX.md` para identificar la plantilla base más cercana al componente que va a construir.

Si en algún punto el agente necesita un método/decorador/campo que no está confirmado en `data/*.csv` del pack ni documentado en `docs/`, debe grepear el core vendorizado indicado en `PRE_CODE_GATE.md` antes de inventar la firma. Si no lo puede verificar, debe decirlo explícitamente en el código (comentario) y en su respuesta al usuario, nunca presentarlo como confirmado.

---

## 1. Context

Este documento define la construcción, **desde cero**, de una integración WordPress/WooCommerce ↔ Odoo 19 bajo una arquitectura de 4 módulos desacoplados, orientada a eventos, sin bloqueos de transacción y con idempotencia garantizada a nivel de base de datos.

El agente de IA que ejecuta este trabajo debe tratar esta especificación como su **única** fuente de verdad arquitectónica. Además de los 4 módulos originales (`integrator_events/`, `integrator_woocommerce/`, `integrator_orchestrator/`, `client_custom_ui/`), `crm_partner_custom_fields/` está **en alcance de forma permanente** (decisión del usuario 2026-08-31): es el módulo que agrega a `res.partner`/`crm.lead` los campos que no existen nativamente en Odoo 19 para este cliente (ver `crm_partner_custom_fields/FIELDS_DOCUMENTATION.md`). Se instala siempre junto a los otros 4, pero mantiene su propia regla de dependencia — ver sección 3.5. Cualquier otro módulo o carpeta que el agente encuentre en el directorio de trabajo por fuera de estos 5 sigue **fuera de alcance** y no debe leerse, citarse, ni usarse como referencia de ningún tipo — ni siquiera como "qué no hacer". Si el agente lo encuentra, lo ignora por completo.

---

## 2. Environment

| Parámetro | Valor |
|---|---|
| Directorio de trabajo (destino del código nuevo) | `C:\Users\Brian\Downloads\fundacionanan-main\fundacionanan-main` |
| Directorio de plantillas (base de conocimiento, solo lectura) | `C:\Users\Brian\Documents\recursos\templates odoo\19` |
| Instancia Odoo 19 Enterprise de desarrollo | Docker, `C:\Users\Brian\Documents\recursos\odoo19`, expuesta en `http://localhost:8070` |
| Versión de Odoo objetivo | `saas~19.2` (`odoo/release.py`: `version_info = ('saas~19', 2, 0, FINAL, 0, '')`) |
| Versión mínima de Python | `3.12` (`MIN_PY_VERSION` del core) |
| Sistema operativo del entorno de desarrollo | Windows (PowerShell) |

**Regla de directorios:** el agente **nunca** escribe ni modifica archivos dentro de `templates odoo\19` (es de solo lectura). Todo el código nuevo se crea directamente bajo la raíz del directorio de trabajo, un addon por carpeta: `integrator_events/`, `integrator_woocommerce/`, `integrator_orchestrator/`, `client_custom_ui/`. Ninguna otra carpeta del directorio de trabajo es relevante para este proyecto (ver sección 1).

---

## 3. Architecture — Sistema de 4 módulos

```
integrator_events            <- núcleo de eventos, no sabe qué es WooCommerce
   ^
   | depends
   |
integrator_woocommerce       <- traductor HTTP <-> evento. No contiene lógica de negocio.
   |
   | (no depende de orchestrator; son pares desacoplados vía la tabla de eventos)
   v
integrator_orchestrator      <- única capa que toca res.partner / crm.lead / sale.order / project.project
   ^
   | depends (solo para decorar vistas de todo lo anterior)
   |
client_custom_ui             <- capa puramente visual, cero lógica de negocio
```

### 3.1 `integrator_events` (núcleo de eventos)

**Responsabilidad única:** recibir, almacenar, deduplicar y reintentar eventos. No sabe qué es WooCommerce, ni qué es un pedido de venta. Es infraestructura pura y reutilizable para cualquier fuente futura (no solo WooCommerce).

- Modelo `app.event.inbound`: tabla buffer (contrato completo en sección 4).
- Idempotencia a nivel SQL por `(event_type, external_id)` usando `models.UniqueIndex(...)` — nunca `_sql_constraints`.
- `ir.cron` de procesamiento y de reintento, con cursores independientes por evento (regla 3, sección 5).
- Expone un método de servicio (`_register_event(...)`) que `integrator_woocommerce` llama para insertar en el buffer — es la única puerta de entrada al modelo.
- **Depende de:** `base`, `mail` (para trazabilidad/chatter si aplica). Nada de `sale`/`crm`/`project`/HTTP externo.

### 3.2 `integrator_woocommerce` (conector)

**Responsabilidad única:** hablar HTTP con WordPress/WooCommerce. Traduce webhooks entrantes a filas en `app.event.inbound` y expone lo necesario para que, más adelante, algo (el orquestador) pueda hacer llamadas salientes hacia la API REST de WooCommerce. No decide qué hacer con un pedido — eso es trabajo del orquestador.

- Controladores HTTP (`type="http"`, `auth="public"`) que reciben webhooks de **Ventas** (`order.*`) y **Formularios** (leads/contacto).
- Validación de firma (HMAC, ver sección 8) **antes** de tocar el ORM.
- Respuesta **HTTP 200 inmediata** tras el `create()` en el buffer — cero procesamiento de negocio en el controlador.
- Modelo de configuración de la conexión (host, credenciales de la REST API de WooCommerce, secreto de firma de webhooks por topic).
- **Depende de:** `integrator_events`, `base`, `web`. **No depende de `sale`, `crm` ni `project`** — si el agente necesita importar esos modelos aquí, es señal de que la lógica pertenece al orquestador, no acá.

### 3.3 `integrator_orchestrator` (lógica de negocio)

**Responsabilidad única:** consumir eventos ya persistidos en `app.event.inbound` (nunca HTTP directo) y orquestar búsqueda/creación en `res.partner`, `crm.lead`, `sale.order`, `project.project`. Es la única capa autorizada a escribir en esos 4 modelos con datos provenientes de WooCommerce.

- Se suscribe a eventos vía el propio cron de `integrator_events` (patrón "handler por `event_type`": un método/dispatcher que resuelve qué hacer según el tipo de evento ya persistido, no polling propio a WooCommerce).
- Búsqueda de partner existente por email/`external_customer_id` antes de crear (evitar duplicados ante reintentos del mismo evento o eventos fuera de orden).
- Reglas de negocio de conversión evento→documento Odoo viven acá y solo acá.
- **Depende de:** `integrator_events`, `sale`, `crm`, `project`, `stock` si corresponde. **No depende de `integrator_woocommerce`** — el orquestador no sabe (ni le importa) que el origen fue WooCommerce; solo conoce la forma del evento en el buffer. Esto es lo que permite, a futuro, otra fuente de eventos sin tocar esta capa.

### 3.4 `client_custom_ui` (capa visual)

**Responsabilidad única:** vistas heredadas, campos puramente estéticos/informativos (badges de origen, colores, iconos), overrides SCSS. **Cero lógica de negocio, cero campos que se usen para tomar decisiones en `integrator_orchestrator`.** Si un campo definido acá termina siendo leído por lógica de negocio, ese campo se movió al módulo equivocado.

- Depende de los módulos cuyas vistas decora (`integrator_orchestrator`, `integrator_woocommerce` si expone alguna vista propia).
- Todo el JS/OWL sigue el patrón de `08_backend_owl_client_action` del template pack (sin encabezado `/** @odoo-module **/`, registro en `registry.category("actions")`).

### 3.5' `crm_partner_custom_fields` (campos de negocio en Contactos/CRM)

**Responsabilidad única:** agregar a `res.partner` y `crm.lead` los campos de negocio que el cliente necesita y que no existen de forma nativa en Odoo 19 (auditado contra el core real, `crm_partner_custom_fields/FIELDS_DOCUMENTATION.md` documenta qué se descartó por duplicado y por qué). Es la capa de "vocabulario de negocio" del CRM/Contactos: `product_interest_id`, `job_position_type`, `conversion_channel`, `organization_type`, etc. Cero lógica de integración con WooCommerce/eventos — eso es trabajo exclusivo de `integrator_orchestrator`, que solo LEE estos campos (nunca al revés).

- Depende de: `base`, `contacts`, `crm`, `product`, `utm`. Nunca de `integrator_events`/`integrator_woocommerce`/`integrator_orchestrator`/`client_custom_ui`.
- Vistas: hereda `base.view_partner_form` y `crm.crm_lead_view_form` directamente (no pasa por `client_custom_ui`, a diferencia del resto del proyecto) — son campos de negocio, no decoración visual, así que viven junto al modelo que los define.

### 3.5 Reglas de dependencia (no negociables)

- `integrator_events` no importa nada de los otros 4.
- `integrator_woocommerce` solo depende de `integrator_events`.
- `integrator_orchestrator` solo depende de `integrator_events` (+ apps core de Odoo). Nunca de `integrator_woocommerce`.
- `client_custom_ui` no contiene modelos de negocio, solo vistas/assets.
- Ningún módulo llama a la API REST de WooCommerce **ni** procesa un webhook fuera de `integrator_woocommerce`.
- `crm_partner_custom_fields` no depende de ninguno de los otros 4 módulos de este proyecto (su manifest solo depende de `base`, `contacts`, `crm`, `product`, `utm`) — es un módulo hermano, no una capa de la cadena de eventos. `integrator_orchestrator` nunca lo declara como dependencia dura (no aparece en su `depends`); cuando necesita leer/escribir un campo que vive ahí (ej. `crm.lead.product_interest_id`, `crm.lead.job_position_type`), lo hace siempre con el mismo chequeo defensivo ya establecido en el código (`"campo" in self._fields` / `hasattr(...)`), nunca asumiendo su presencia. Esto es intencional: permite instalar/desinstalar `crm_partner_custom_fields` sin romper la carga de `integrator_orchestrator`, aun estando ambos siempre instalados juntos en este proyecto en la práctica.

---

## 4. Data Contract — `app.event.inbound`

Modelo base a construir extendiendo el patrón ya verificado de `10_connector_sync_no_queue/x_template_connector_sync_binding.py` (idempotencia) + `05_cron_batch_idempotent/x_template_cron_batch_job.py` (máquina de estados de reintento). No reinventar estos dos patrones — combinarlos.

```python
# integrator_events/models/app_event_inbound.py
class AppEventInbound(models.Model):
    _name = "app.event.inbound"
    _description = "Inbound integration event buffer"
    _order = "id desc"

    source = fields.Selection([("woocommerce", "WooCommerce")], required=True, index=True)
    event_type = fields.Char(required=True, index=True)        # ej: "order.created", "form.submitted"
    external_id = fields.Char(required=True, index=True)        # id del recurso en el sistema origen
    payload_json = fields.Text(required=True)
    payload_hash = fields.Char(index=True)                      # sha256 del payload normalizado

    state = fields.Selection(
        [("pending", "Pending"), ("processing", "Processing"), ("done", "Done"), ("error", "Error")],
        default="pending", required=True, index=True,
    )
    attempts = fields.Integer(default=0, readonly=True)
    next_attempt_at = fields.Datetime(index=True)
    last_error = fields.Text(readonly=True)
    processed_at = fields.Datetime(readonly=True)

    # Idempotencia SQL real — nunca _sql_constraints (dead en saas~19.2, ver sección 6)
    _event_external_uniq = models.UniqueIndex("(source, event_type, external_id)")
```

Reglas de este modelo:

- El `create()` que inserta acá (llamado desde `integrator_woocommerce`) debe ser **la única escritura** — si el registro ya existe por el índice único, el webhook responde 200 igual (WooCommerce reintenta webhooks agresivamente; un duplicado no debe romper nada, debe ser un no-op silencioso y logueado).
- `payload_json` se guarda tal cual llega (crudo), nunca parcialmente parseado — el parseo/mapeo a campos de negocio ocurre en `integrator_orchestrator`, no acá.
- `payload_hash` permite distinguir "mismo `external_id`, payload distinto" (ej. un pedido que cambió de estado) de un reintento idéntico de WooCommerce.

---

## 5. Reglas operativas (obligatorias)

### Regla 1 — Prioridad de plantillas
Antes de escribir la estructura de cualquier modelo, controlador o vista, el agente **debe** leer la carpeta correspondiente en `templates odoo\19\odoo19_saas192_template_pack\templates\` y anunciar en su respuesta qué plantilla usó como base y por qué (según `TEMPLATE_MATRIX.md`). Si ninguna plantilla calza, debe decirlo explícitamente antes de improvisar, no en silencio.

Mapeo directo para este proyecto:

| Componente a construir | Plantilla base |
|---|---|
| `app.event.inbound` + cron de procesamiento | `05_cron_batch_idempotent` |
| Modelo/estructura de idempotencia por `external_id` | `10_connector_sync_no_queue` (patrón `binding` + `UniqueIndex`) |
| Controlador de webhook (público, con firma) | `06_json_controller` (ruta pública tokenizada: validar antes de `sudo()`) |
| Modelo interno simple con vista/menú/seguridad | `01_minimal_business_module` |
| Campos nuevos sobre `res.partner`/`sale.order` | `02_model_extension_smart_button` |
| Estado/workflow en `crm.lead` u orquestación con chatter | `03_mail_thread_approval` |
| `res.config.settings` de la conexión WooCommerce | `09_res_config_settings` |
| Vistas/JS de `client_custom_ui` | `08_backend_owl_client_action` |

### Regla 2 — Cero middlewares
Toda la ingesta y orquestación ocurre **dentro de Odoo**. No se sugiere, no se instala y no se referencia en ningún manifest: FastAPI, Celery, n8n, colas externas (Redis/RabbitMQ) ni tampoco `queue_job` (OCA) — el pack incluye una plantilla `13_optional_oca_queue_job` pero está **fuera de alcance** para este proyecto: la regla 3 (cursores independientes) es el reemplazo funcional de `queue_job` que este proyecto usa. Si el agente considera que hace falta una cola externa, debe plantearlo como pregunta al usuario, nunca implementarlo por iniciativa propia.

### Regla 3 — Manejo de transacciones en los crons
En todo cron de `integrator_events` que procese múltiples eventos, cada evento se procesa en **su propio cursor**, aislado del cursor del cron:

```python
@api.model
def _cron_process_pending_events(self, limit=100):
    events = self.search([...], limit=limit)
    for event in events:
        try:
            with self.env.registry.cursor() as new_cr:
                new_env = api.Environment(new_cr, self.env.uid, self.env.context)
                new_env["app.event.inbound"].browse(event.id)._process_one()
                # salida limpia del `with` -> auto-commit de este cursor (verificado
                # en odoo/sql_db.py, Cursor.__exit__: commitea si no hubo excepción).
        except Exception as error:
            _logger.exception("Event %s failed: %s", event.id, error)
            # el cursor de arriba se descartó sin commit porque la excepción salió
            # del `with` antes de llegar acá (Cursor.__exit__ no commitea si hay
            # excepción). Un cursor nuevo y corto registra el error.
            with self.env.registry.cursor() as err_cr:
                err_env = api.Environment(err_cr, self.env.uid, self.env.context)
                err_env["app.event.inbound"].browse(event.id)._mark_error(str(error))
    return True
```

Importante: el `try/except` envuelve el bloque `with` completo, no queda anidado adentro. Si se captura la excepción *dentro* del `with`, este sale "limpio" y comitea igual una transacción que puede haber quedado abortada a nivel Postgres — hay que dejar que la excepción escape del `with` para que no comitee.

Justificación: el fallo (excepción, timeout de red hacia WooCommerce, lock de fila) de **un** evento no debe hacer rollback de los eventos ya procesados en la misma corrida ni bloquear el resto de la cola. Nunca usar `self.env.cr.commit()` a secas dentro de un bucle — o se aísla con cursor propio, o no se hace commit parcial.

### Regla 4 — Metodología de desarrollo (el agente como tutor)
- El agente **no sobreescribe archivos masivamente a ciegas**. Cada componente (un modelo, un controlador, una vista) se genera, se explica y se somete a validación del usuario antes de continuar con el siguiente.
- Al proponer un componente, el agente explica: qué plantilla usó, qué trade-off tomó (ej. "cursor independiente por evento agrega overhead de conexión pero aísla fallos — alternativa: procesar en batch con un solo cursor y aceptar que un fallo tumbe el batch, descartada por la regla 3"), y qué falta validar manualmente antes de dar el componente por cerrado.
- Nunca generar los 4 módulos completos en una sola pasada. Orden sugerido: `integrator_events` → `integrator_woocommerce` (solo ingesta) → `integrator_orchestrator` → `client_custom_ui`.
- Si el usuario pide "generá todo el módulo X", el agente igual debe presentarlo dividido por archivo/componente en su explicación, aunque escriba varios archivos en la misma respuesta — la exigencia es sobre la trazabilidad y explicación, no sobre la cantidad de tool calls.

### Regla 5 — Calidad de código
- PEP8.
- ORM eficiente: `search_fetch`, `_read_group` para agregaciones, `mapped`/`filtered` con criterio (no como excusa para traer una tabla completa a memoria).
- Cero bucles `for` con `search()`/`write()`/llamada HTTP individual dentro cuando se puede resolver en lote (ver checklist de performance en sección 6).
- `@api.model_create_multi` en todo `create()` override, recibiendo `vals_list`.
- `Command.*` para toda operación sobre campos x2many.
- Sin decoradores/atributos obsoletos (ver tabla de la sección 6).

---

## 6. Contrato de API Odoo 19 (`saas~19.2`) — Referencia rápida

Esta tabla resume `SOURCE_PATTERNS.md` y `docs/03_changes_from_odoo17.md` del pack. Es información **verificada contra el core real**, no supuesta. Ante cualquier duda que no esté acá, aplicar el protocolo de `PRE_CODE_GATE.md` (grep contra el core vendorizado) antes de escribir.

| En vez de (Odoo 17 / hábito viejo) | Usar en este proyecto (saas~19.2, verificado) |
|---|---|
| `_sql_constraints = [(...)]` | `models.Constraint("...", "mensaje")` / `models.Index("...")` / `models.UniqueIndex("...")` como atributo de clase, nombre iniciado en `_` |
| `type="json"` en rutas de controller | `type="jsonrpc"` |
| `check_access_rights()` / `check_access_rule()` | `check_access(operation)` / `has_access(operation)` |
| `attrs="{...}"` / `states="{...}"` en XML | Modificadores por expresión directa: `invisible="state != 'draft'"` |
| `<tree>` en vistas backend | `<list>` |
| `create(self, vals)` simple | `@api.model_create_multi` + `create(self, vals_list)` |
| Escribir x2many con tuplas `(0, 0, {...})` | `Command.create({...})` / `Command.set([...])` / etc. |
| Construir dominios como strings/concatenación | Lista de tuplas clásica para dominios estáticos; `Domain`/`Domain.AND([...])` (`odoo/orm/domains.py`) solo cuando el dominio se arma programáticamente por partes |
| `<group expand="0" string="Group By">` para la sección de agrupado en una vista `search` | `<group name="group_by">` a secas — `expand`/`string` en ese `<group>` ya no valida contra el esquema RNG en 19 (verificado: rompe el load del módulo con `RELAXNG_ERR_INVALIDATTR`; confirmado el patrón correcto contra `addons/base/views/res_partner_views.xml`) |

Metadata de manifest fija para los 4 módulos de este proyecto (no placeholder, usar siempre estos valores):

```python
"author": "Brian Escalante",
"website": "https://www.linkedin.com/in/brian-escalante/",  # link de más información, por ahora
"license": "OPL-1",  # Odoo Proprietary License — licencia de pago, no LGPL-3
```

Reglas de estilo del pack que también aplican acá (`docs/02_development_standard.md`):

- Orden de archivos por módulo: `models/ → wizard/ → controllers/ → security/ → data/ → views/ → report/ → static/ → tests/`.
- Orden de carga en el manifest: grupos → access CSV → record rules → data técnica → wizards → reports → vistas → menús al final.
- `sudo()` siempre acotado y comentado con la razón — nunca "porque si no, no funciona".
- Todo dato crítico (credenciales, secretos de firma) protegido por `groups=` en el campo, nunca visible para el grupo de usuario estándar.

---

## 7. Anti-patrones prohibidos en este proyecto

El agente debe tratar cada fila como una regla absoluta de "no hacer esto", con su corrección obligatoria.

| Antipatrón prohibido | Corrección obligatoria |
|---|---|
| Cron de alta frecuencia haciendo N llamadas HTTP síncronas dentro del propio bucle de negocio | Ingesta por webhook (push, no polling) + procesamiento por evento con cursor aislado (regla 3) |
| Reimplementar a mano, dentro de un loop, un cálculo agregado que el ORM ya expone, sin distinguir "no hay dato" de "el valor es 0" | Usar los campos/métodos que el ORM ya expone; validar explícitamente ausencia de dato antes de persistir un valor por defecto |
| `self.env.cr.commit()` suelto dentro de un `for`, sin aislar el error de una iteración del resto | Cursor independiente por evento (regla 3), nunca commit manual sobre el cursor compartido del cron |
| Override completo de una ruta/controller core sin llamar a `super()`, reimplementando el flujo original | Ningún controller de este proyecto reimplementa flujos core; solo rutas nuevas y propias bajo su propio namespace |
| `_sql_constraints` (sintaxis muerta en 19.2) | `models.UniqueIndex(...)` / `models.Constraint(...)` (ver sección 4 y 6) |
| Agregar campos a modelos core sin evaluar el impacto de una futura desinstalación | Cada módulo que agregue campos a modelos core (`res.partner`, `sale.order`, etc.) debe evaluar si necesita `uninstall_hook`; documentarlo explícitamente aunque la decisión sea "no hace falta" |
| Modelar infraestructura de cola/buffer y luego no usarla realmente desde la lógica de negocio | `app.event.inbound` es la única cola y **todo** el flujo pasa por ella, sin atajos directos modelo-a-modelo |
| Bloquear la transacción de una acción de usuario (validar un albarán, confirmar una orden) esperando respuesta HTTP de un sistema externo | Ninguna acción de usuario en Odoo espera una respuesta HTTP externa; toda salida hacia WooCommerce se desacopla vía evento/cron |
| Credenciales de API en texto plano, visibles/editables por un único grupo de permisos sin distinción operador/admin | Campos de secreto con `groups="base.group_system"` (ver patrón en `10_connector_sync_no_queue/x_template_connector_sync_backend.py:21`, campo `api_key`); grupo operativo separado del grupo con acceso a configuración |

---

## 8. Security Contract

### 8.1 Webhooks entrantes (`integrator_woocommerce`)

- Ruta pública: `type="http"`, `auth="public"`. **Nunca `auth="none"`** salvo justificación explícita.
- WooCommerce firma cada webhook con HMAC-SHA256 en base64 del cuerpo crudo de la request, usando el secreto configurado por topic, enviado en el header `X-WC-Webhook-Signature`. Esto es contrato de WooCommerce, no del core de Odoo — no está en `PRE_CODE_GATE.md`; el agente debe tratarlo como dato externo verificado por documentación de WooCommerce, y decirlo así si lo cita.
- La validación de firma ocurre **antes** de cualquier `sudo()` y antes de tocar el ORM, comparando en tiempo constante (nunca `==` para comparar firmas — timing attack). Usar `odoo.tools.consteq` (`from odoo.tools import consteq`) — verificado en el core real (`odoo/tools/misc.py`: `consteq = hmac_lib.compare_digest`) y es el mismo helper que usa `pos_platform_order_gofood/controllers/main.py` del propio core para validar firma de webhook de una plataforma externa; ese archivo es la referencia idiomática más cercana a lo que hace `integrator_woocommerce`, incluyendo el patrón de idempotencia con `cr.savepoint()` + `except psycopg2.errors.UniqueViolation` (ver sección 4, `_register_event`).
- Si la firma no valida: responder 401 y loguear el intento (sin loguear el payload completo si contiene datos personales sensibles) — no crear evento.
- Si la firma valida: `sudo().create()` en `app.event.inbound` con el payload crudo, commit, responder 200 inmediato. Cero lógica de negocio en el controller.

### 8.2 Secretos

- Ningún secreto (consumer key/secret de la REST API de WooCommerce, secreto de firma de webhook) se loguea, ni completo ni parcial, en `_logger` ni en ningún chatter/log de negocio.
- Todo campo de secreto: `groups="base.group_system"` como mínimo, considerar cifrado a nivel de campo si el usuario lo pide explícitamente (no asumir que hace falta sin confirmarlo).

### 8.3 Salidas hacia WooCommerce (llamadas REST desde `integrator_orchestrator`/`integrator_woocommerce`)

- Timeout explícito y razonable en toda llamada saliente (no usar el default de la librería sin fijarlo).
- Manejo explícito de 429/5xx con backoff, nunca reintento inmediato en loop cerrado.
- Verificación de certificado TLS siempre activa (nunca `verify=False`).

---

## 9. Workflow (bucle de trabajo del agente por componente)

Para cada componente nuevo (un modelo, un controller, un cron, una vista):

1. **Anunciar el plan corto**: qué se va a construir, qué plantilla de la sección 5/Regla 1 se usa, en qué archivo(s).
2. **Verificar API** contra `PRE_CODE_GATE.md` si algo no está 100% seguro.
3. **Escribir el componente** (no más de lo anunciado en el paso 1).
4. **Explicar el trade-off** relevante de esa pieza, si lo hay (rendimiento vs. simplicidad, sync vs. async, alcance del `sudo()`, etc.).
5. **Listar qué falta validar manualmente** antes de considerarlo terminado (ej. "falta correr `-u integrator_events --test-enable --stop-after-init` contra una instancia real").
6. **Detenerse y esperar validación del usuario** antes de encadenar el siguiente componente, salvo que el usuario haya pedido explícitamente avanzar sin pausas para un lote de trabajo acotado.

No se generan los 4 módulos de punta a punta sin checkpoints intermedios, incluso si el usuario pide "hacé todo el flujo": el agente puede trabajar rápido, pero siempre mostrando el corte por componente en su respuesta.

---

## 10. Definition of Done (checklist de aceptación por módulo)

Basado en `checklists/before_install.md` y `checklists/code_review_odoo19.md` del pack, aplicado a este proyecto:

- [ ] Dependencias del manifest mínimas y completas (ver reglas de dependencia, sección 3.5).
- [ ] `security/*.xml` (grupos) carga antes que `ir.model.access.csv`.
- [ ] ACL explícita para todo modelo persistente y transitorio con acceso desde UI.
- [ ] Ningún `_sql_constraints`; solo `models.Constraint`/`models.Index`/`models.UniqueIndex`.
- [ ] Ninguna ruta de controller con `type="json"`.
- [ ] Ningún `attrs`/`states` en XML nuevo; vistas de lista usan `<list>`.
- [ ] Ningún `check_access_rights()`/`check_access_rule()`; solo `check_access()`/`has_access()`.
- [ ] `create()` con `@api.model_create_multi`.
- [ ] Ningún `ensure_one()` dentro de `write()`/`unlink()`.
- [ ] Ningún `sudo()` sin comentario explicando el alcance validado.
- [ ] Crons idempotentes, acotados por `limit`, con cursor aislado por evento cuando iteran sobre `app.event.inbound` (regla 3).
- [ ] Ninguna llamada HTTP saliente dentro de un método invocado directamente por una acción de usuario en el hilo principal (botones, `button_validate`, etc.) — siempre vía evento/cron.
- [ ] Índice único `(source, event_type, external_id)` presente y probado (insertar el mismo evento dos veces no duplica ni rompe).
- [ ] Tests: al menos un test por módulo que cree un registro y corra el camino principal (patrón de `tests/test_*.py` de cada plantilla usada).
- [ ] Sin secretos en logs (grep manual de `_logger` antes de cerrar el componente).

---

## 11. Apéndice — Índice de plantillas y documentos usados como base

Todos los paths son relativos a `C:\Users\Brian\Documents\recursos\templates odoo\19\`.

- `odoo19_saas192_template_pack\TEMPLATE_MATRIX.md` — selección de plantilla por necesidad.
- `odoo19_saas192_template_pack\SOURCE_PATTERNS.md` — anclas verificadas contra el core real de saas~19.2.
- `odoo19_saas192_template_pack\USAGE.md` — prompt base recomendado por el propio pack para generación asistida.
- `odoo19_saas192_template_pack\checklists\before_install.md`, `code_review_odoo19.md`, `template_adaptation.md`.
- `odoo19_saas192_template_pack\templates\05_cron_batch_idempotent\` — base de `app.event.inbound` (máquina de estados de reintento).
- `odoo19_saas192_template_pack\templates\10_connector_sync_no_queue\` — base de idempotencia por `external_id` (`UniqueIndex`, patrón backend/binding).
- `odoo19_saas192_template_pack\templates\06_json_controller\` — base del controller de webhook (ruta pública + validación antes de `sudo()`).
- `odoo19_saas192_template_pack\templates\01_minimal_business_module\`, `02_model_extension_smart_button\`, `03_mail_thread_approval\`, `09_res_config_settings\`, `08_backend_owl_client_action\` — bases del resto de componentes según tabla de la Regla 1.
- `odoo19_dev_docs\PRE_CODE_GATE.md` — protocolo de verificación de API antes de codear.
- `odoo19_dev_docs\docs\02_development_standard.md` — estándar de estilo/estructura de módulo.
- `odoo19_dev_docs\docs\03_changes_from_odoo17.md` — deltas conocidos 17→19.
- `odoo19_dev_docs\docs\anti_patterns\` (`architecture_anti_patterns.md`, `orm_anti_patterns.md`, `security_anti_patterns.md`, `xml_anti_patterns.md`) — consultar ante cualquier duda de diseño no cubierta acá.
- `odoo19_dev_docs\docs\cookbook\crons.md`, `controllers.md`, `model_inheritance.md` — recetas puntuales por tema.
- `odoo19_dev_docs\docs\checklists\module_development_checklist.md`, `code_review_checklist.md` — checklist complementaria a la sección 10 de este documento.
