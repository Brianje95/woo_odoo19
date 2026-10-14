# Flujo de negocio — integración WooCommerce ↔ Odoo

> Este documento describe **qué hace cada cosa y en qué orden**, en tiempo de ejecución. No es lo mismo que `agent_spec.md` (que define reglas de arquitectura/estilo de código para quien programa esto) — acá se documenta el comportamiento funcional real, para no tener que reconstruirlo leyendo el código o el historial de chat cada vez.

Si cambia una regla de negocio, este archivo se actualiza en el mismo commit que el código.

---

## 1. Visión general

```
WooCommerce (tienda)
   │ webhook (push, tiempo real)              │ cron cada 15 min (red de seguridad)
   │ POST /integrator_woocommerce/webhook     │ woo.connection._woo_check_new_orders()
   ▼                                          ▼
              app.event.inbound (buffer, estado "pending")
                          │
                          │ se procesa solo, apenas se guarda (postcommit,
                          │ agent_spec.md Regla 3 -- cursor propio aislado);
                          │ cron "Woo: process pending events" cada 5 min
                          │ queda como red de seguridad (reintentos, o algún
                          │ evento que por lo que sea no se disparó solo)
                          ▼
          integrator_orchestrator._process_one() → _handle_<event_type>
                          │
        ┌─────────────────┼──────────────────────┐
        ▼                 ▼                      ▼
  order_created     order_cancelled         order_updated
  contacto +        cancela la orden        (sin implementar
  oportunidad       de venta si             todavía)
  (type=opportunity) corresponde

Oportunidad → etapa "Ganado" (is_won=True, cualquiera)
        │
        ▼
  sale.order: se crea, se confirma si tiene líneas → dispara proyecto/tarea
  (sale_project) + mail "Hemos recibido tu pedido"
```

---

## 2. La conexión (`woo.connection`)

Un registro por tienda WooCommerce. Campos relevantes:

- `store_url`, `company_id`.
- `consumer_key` / `consumer_secret`: **solo** para llamadas salientes de Odoo hacia WooCommerce (sync de catálogo, polling de pedidos nuevos/cancelados). No tienen nada que ver con la autenticación de los webhooks entrantes.
- `webhook_subscription_ids`: una línea por topic suscripto (`order.created`, `order.updated`, etc.), cada una con **su propio** secret.
- No tiene ningún identificador en la URL del webhook (ni slug ni token) — ver sección 3.

---

## 3. Cómo identifica Odoo de qué tienda es un webhook entrante

URL fija y única para todas las tiendas: `POST /integrator_woocommerce/webhook`.

1. Si el request no trae el header `X-WC-Webhook-Topic` → es el ping que manda WooCommerce al guardar un webhook. Se responde 200 sin crear nada ni validar firma (el ping no lleva firma).
2. Si trae topic, Odoo prueba la firma (`X-WC-Webhook-Signature`, HMAC-SHA256 en base64) contra el secret de **cada línea** que tenga ese mismo topic, en cualquier conexión activa, hasta encontrar una coincidencia (`odoo.tools.consteq`, comparación en tiempo constante). Ninguna coincide → 401.
3. La conexión/tienda se deduce de **cuál secret matcheó** — no hay ningún otro identificador en juego.

Esto significa: cada topic tiene su propio secret por tienda, no se comparte un único secret por conexión.

---

## 4. Ingesta de pedidos nuevos — 3 caminos al mismo buffer

| Camino | Cuándo | Qué hace |
|---|---|---|
| Webhook | Tiempo real, apenas se crea el pedido | `order.created` llega por POST, se valida firma, se registra el evento |
| Cron "Buscar pedidos nuevos" | Cada 15 min | Pide a la API REST los pedidos de una ventana móvil (ver abajo), registra un evento por cada uno |
| Botón "Check New Orders" | Manual, en la conexión | Mismo código que el cron, para forzarlo ya |

Los tres terminan en `app.event.inbound._register_event(source, event_type, external_id, payload_json)`, que es **idempotente** por `(source, event_type, external_id)` — reprocesar el mismo pedido nunca duplica nada, sea por el camino que sea. Todo evento queda vinculado a su conexión (`woo_connection_id`), sin importar por cuál de los tres caminos llegó.

El cron de pedidos nuevos existe como red de seguridad: si el webhook falla por lo que sea (proxy caído, túnel de ngrok con URL vieja, webhook mal configurado en WooCommerce), el pedido no se pierde para siempre — como máximo tarda 15 minutos en aparecer.

**Ventana de búsqueda:** el cron/botón no miran "desde la última revisión exacta" — eso perdía para siempre un pedido si alguien borraba su evento a mano después de que la marca de agua ya había avanzado. Miran siempre una **ventana móvil de los últimos 30 días** (`WOO_ORDER_LOOKBACK_DAYS`), así un evento borrado por error se vuelve a traer solo en la próxima corrida. Los campos `last_order_check`/`last_cancelled_check` en la conexión son solo informativos, ya no acotan la búsqueda.

**Consulta puntual (Desde/Hasta):** en la conexión hay dos campos de fecha, `Desde`/`Hasta`, vacíos por defecto. Si se completan, la próxima vez que se toque "Check New Orders" (o "Check Opportunities", sección 4.1) usa ese rango puntual en vez del comportamiento por defecto. Apenas termina esa corrida, los dos campos se vacían solos — la vez siguiente vuelve a comportarse como siempre, sin que haga falta limpiarlos a mano.

Existe el mismo patrón de ventana móvil para pedidos **cancelados** (sección 8), con su propio cron horario y su propio botón.

---

### 4.1 Reponer oportunidades borradas por error — botón "Check Opportunities"

Si alguien borra una **oportunidad de CRM** (o la orden de venta) directamente en Odoo -- no el evento -- ni "Check New Orders" ni el webhook la van a traer de vuelta: el evento (`app.event.inbound`) sigue existiendo y en estado `done`, así que para el sistema ese pedido ya está "resuelto"; los caminos de la sección 4 solo van a buscar a WooCommerce lo que **todavía no está** en el buffer de Odoo.

Para este caso está el botón **"Check Opportunities"** ("Consultar oportunidades"), en la misma conexión. A diferencia de todo lo demás en este documento, **no llama a la API de WooCommerce** — reprocesa en masa (un solo clic, nunca uno por uno) los eventos `order_created` que ya están guardados y en estado `done` para esa conexión, volviendo a correr el mismo dispatch genérico (`_process_one` → `_handle_order_created`) que ya tiene su propia idempotencia: si la oportunidad ya existe, no hace nada; si no existe (porque se borró), la vuelve a crear.

- Sin Desde/Hasta completos: repasa **todo** el historial de eventos guardados de esa conexión — sirve para un desastre grande (se borraron varias oportunidades de una).
- Con Desde/Hasta completos (mismos campos de la sección 4): solo repasa los eventos cuyo pedido tenga fecha de creación dentro de esa ventana — el filtro se aplica sobre la fecha real del pedido (dentro del `payload_json` guardado), no sobre cuándo Odoo registró el evento.
- Este botón nunca llama a WooCommerce ni sabe nada de `crm.lead`/`sale.order` a nivel de código (`integrator_woocommerce` no depende de `integrator_orchestrator`) — solo dispara el mismo mecanismo genérico que ya usa el cron de procesamiento del buffer (sección 5).

---

## 5. Procesamiento del buffer (`app.event.inbound`)

- **Disparo automático apenas se guarda el evento** (agregado 2026-08-24): `_register_event` programa el procesamiento de ESE evento (`_process_one_isolated`, cursor propio) para que corra apenas se confirma el `create()` en base — no hace falta esperar al cron para el camino feliz. Aplica a los 3 caminos de la sección 4 por igual (todos pasan por `_register_event`).
- Cron "Woo: process pending events" cada 5 minutos, hasta 100 eventos por corrida, **cursor aislado por evento** — queda como red de seguridad para reintentos (evento que falló y espera su backoff) o algún evento que por lo que sea no se haya disparado solo (ej. reinicio del servidor justo entre el `create()` y el disparo). Si uno falla no hace rollback de los que ya se procesaron bien en la misma corrida.
- 3 intentos por evento (`MAX_ATTEMPTS`), con 15 minutos de espera entre reintentos (`RETRY_BACKOFF_MINUTES`). Al tercer fallo el evento pasa a estado `error` y **deja de reintentarse solo** — hay que revisarlo a mano (campo `last_error`).
- El dispatch es dinámico por `event_type`: busca un método `_handle_<event_type>` en `app.event.inbound`. Si no existe, error explícito (nunca falla en silencio ni se ignora).
- **No hay ningún cron/proceso que borre eventos viejos.** Son el buffer/fuente de verdad de cada pedido — si se borra un evento a mano, se pierde la única copia del detalle de ese pedido (productos, precios, datos de facturación) hasta que se vuelva a traer de WooCommerce.

---

## 6. `order_created` → contacto + oportunidad

### 6.1 Contacto (`res.partner`)

Cascada de búsqueda, en este orden — la primera que matchea gana y ahí termina la búsqueda:

1. `woo_customer_id` exacto (los invitados mandan `customer_id=0`, que nunca cuenta como id real).
2. Email, normalizado y case-insensitive.
3. Teléfono, comparando solo dígitos.

Si ninguna matchea, se crea un contacto nuevo con todos los datos de facturación del pedido.

**Excepción — pedido sin ningún dato de identidad** (sin nombre, sin email, sin teléfono; típico de un test o un checkout roto): en vez de crear un contacto genérico nuevo por cada pedido así, se busca/reutiliza **un único** contacto compartido llamado "WooCommerce Customer" (sin email/teléfono/padre). Nunca se duplica.

Si matchea uno **ya existente**: nunca se pisa un dato que ya tenga cargado, pero si algún campo está vacío (teléfono, calle, ciudad, código postal, país, provincia) se completa con lo que trae el pedido nuevo.

**Si ninguno de los tres matchea** (es un comprador nuevo), antes de crear un contacto suelto se prueba primero si corresponde a una empresa ya cargada — ver sección 6.1bis.

Todo contacto nuevo que crea este flujo (el contacto suelto de acá, el contacto hijo B2B de 6.1bis, y la dirección de facturación de 6.3) queda etiquetado con la etiqueta configurable de la sección 6.1ter, para poder identificar a simple vista qué contactos se originaron en una compra web.

**Idioma por defecto** (agregado 2026-08-31): todo contacto nuevo que crea este flujo (el mismo alcance del párrafo anterior) queda en `es_AR` (Español, Argentina) si ese idioma está instalado/activo en la base — nunca se fuerza si no lo está, y nunca se pisa el idioma de un contacto ya existente que se matchea por email/teléfono/`woo_customer_id`.

**Provincia (`state_id`)**: igual que el tipo de identificación CUIT/DNI, nunca texto libre — siempre se matchea contra la lista real de provincias de Odoo (`res.country.state`), scopeada al país del pedido. Primero por código (WooCommerce manda el mismo código corto tipo ISO-3166-2 que ya usa Odoo para Argentina, ej. "S" = Santa Fe), si no matchea por código, por nombre completo. Si no matchea ninguno de los dos, queda sin setear y se avisa por nota interna al vendedor asignado (mismo mecanismo que un CUIT inválido) — nunca bloquea el pedido. Aplica tanto al cliente padre como al contacto hijo de facturación (sección 6.3).

**Este contacto (el "cliente padre") ya no recibe CUIT ni condición fiscal directamente** — esos datos son de la *dirección de facturación* del pedido puntual, no de la cuenta del cliente en general. Ver sección 6.3.

**La dirección de facturación (6.3) se crea/busca en el mismo momento que este contacto** — al procesar `order_created`, no recién cuando la oportunidad se gana. Así todo cliente creado por WooCommerce ya tiene su contacto de facturación desde el primer momento, sin depender de que alguien gane la oportunidad después.

### 6.1bis Empresa por dominio de email (contacto hijo B2B)

Requerimiento del cliente, confirmado 2026-08-19. Cuando un comprador nuevo (no matcheó por `woo_customer_id`/email/teléfono en 6.1) tiene un email cuyo dominio coincide con el dominio del email de una **empresa** ya cargada en Odoo (`res.partner`, `is_company = True`), no se crea un contacto suelto — se resuelve así:

1. **Dominio de la empresa**: se toma del campo Email de la ficha de la empresa (soporta lista separada por comas, ver punto 3). El sitio web (`website`) no se usa como fuente. Dominios de correo genéricos (Gmail, Hotmail, Outlook, Yahoo, iCloud, etc.) quedan **excluidos** de este matching a propósito — si la empresa tuviera cargado un email genérico, cualquier comprador no relacionado con esa cuenta de correo terminaría absorbido como su "contacto hijo". Es una decisión técnica, no un pedido explícito del cliente — revisar si en algún caso real hace falta permitir dominios genéricos.
2. **Mismo contacto que la empresa** (no se crea nada nuevo): si el email, el teléfono, o el CUIT/DNI que trae el pedido coincide exactamente con el de la empresa.
   - Coincide el email exacto → no se toca nada (ejemplo del cliente: alguien que compra con el mismo mail de la empresa).
   - Coincide el CUIT/DNI (mismo campo/normalización que la sección 6.3) pero el email es distinto → se agrega el mail nuevo al campo Email de la empresa, separado por coma (ej. `company@odoo.com` → `company@odoo.com, m.angeles@odoo.com`). Nunca se pisa el valor existente, solo se agrega. El teléfono, a diferencia del email, nunca se concatena (es un campo de valor único formateado E.164, no una lista) — si coincide el CUIT pero el teléfono es distinto, ese teléfono nuevo simplemente no se guarda en ningún lado.
   - En cualquiera de los dos casos, la oportunidad queda a nombre de la empresa directamente (`partner_id` = la empresa). La dirección de facturación de 6.3 tampoco se crea en este caso aunque el pedido traiga CUIT: si el CUIT del pedido ya es el mismo que la empresa tiene cargado, se factura directo a la empresa (nunca se crea un hijo de facturación redundante para la misma identidad fiscal).
3. **Contacto hijo nuevo** (ninguno de los tres coincide): se crea un contacto hijo (`parent_id` = la empresa, `type = "contact"`, **no** `"invoice"` — ese tipo es solo para la dirección de facturación de 6.3) con los datos del comprador. No se le carga un CUIT/DNI propio directamente — pero al ser un hijo normal de una empresa, Odoo le sincroniza el CUIT de la empresa automáticamente (igual que cualquier contacto de una empresa en Odoo estándar); eso es solo informativo en su ficha, nunca se usa para decidir a quién facturar (eso siempre pasa por `commercial_partner_id`/la dirección de facturación de 6.3). La oportunidad queda a nombre de este contacto hijo.

**Facturación cuando el comprador es un contacto hijo B2B**: la dirección de facturación por CUIT (sección 6.3) se busca/crea colgada de la **empresa** (`commercial_partner_id`, campo nativo de Odoo que resuelve la empresa de cualquier contacto hijo), no del contacto hijo — así la factura siempre queda a nombre de la empresa padre, tenga o no CUIT propio ese pedido puntual. Para cualquier contacto sin empresa asociada (la gran mayoría de los pedidos), `commercial_partner_id` es el contacto mismo, así que el comportamiento de 6.3 no cambia.

### 6.1ter Etiqueta de contacto configurable

Todo contacto que genera este módulo (contacto suelto de 6.1, contacto hijo B2B de 6.1bis, dirección de facturación de 6.3) se etiqueta con una etiqueta de contacto (`res.partner.category`, la funcionalidad nativa de "Etiquetas" de Contactos). El nombre de la etiqueta es configurable por empresa en el menú de la integración → Configuración (`res.company.woo_customer_tag_name`), por defecto **"COMPRA WOOCOMMERCE"** — el cliente puede cambiarlo a gusto (ej. "LEAD WOO") sin tocar código; el próximo contacto que se cree usa el nombre que esté configurado en ese momento (busca una etiqueta existente con ese nombre exacto, o la crea si no existe).

### 6.3 Dirección de facturación (contacto hijo, por CUIT)

El bloque de facturación de WooCommerce (nombre/razón social, dirección, CUIT, condición fiscal) puede representar una identidad fiscal **distinta** de quien hace la compra — por ejemplo, alguien comprando en nombre de una empresa, o un cliente que factura bajo más de un CUIT según el pedido. Por eso se modela como un **contacto hijo** del cliente padre (tipo "Dirección de facturación", `type=invoice` en Odoo), nunca escribiendo el CUIT directamente sobre el padre.

Búsqueda **acotada a los hijos de ese cliente puntual** (nunca una búsqueda global de CUIT en toda la base):

1. Se normaliza el número sacando puntos, guiones y espacios, y también un cero a la izquierda si lo trae (`vat_digits`, campo indexado, mismo patrón que `phone_digits`) — así "01234567" y "1234567" matchean el mismo contacto.
2. Si ya existe un hijo con ese número normalizado → se reutiliza (nombre/dirección no se tocan, ya están cargados).
3. Si no existe → se crea un contacto hijo nuevo: nombre (razón social del pedido, o nombre y apellido si no hay razón social), dirección, teléfono. **Tipo de identificación según la cantidad de dígitos, solo si el pedido factura a Argentina** (`billing.country == "AR"` en el payload — en la práctica este meta dato de WooCommerce solo lo llena un plugin de checkout específico para Argentina, pero igual se verifica el país por las dudas, confirmado con el usuario 2026-08-16): 7-8 dígitos → DNI (`l10n_ar.it_dni`; el campo de WooCommerce dice "cuit" pero un número de esa longitud es un DNI, no un CUIT mal cargado), cualquier otra longitud → CUIT (`l10n_ar.it_cuit`). Si el pedido no factura a Argentina, no se toca el tipo de identificación (queda el que Odoo asigna por defecto). Nunca queda el genérico "NIF" para un contacto argentino — cada tipo valida con su propio formato (CUIT con dígito verificador, DNI sin él), así que asignar el tipo equivocado rechaza un número real como si fuera inválido. Un número inválido para su tipo nunca bloquea el resto del flujo: el tipo de identificación queda seteado igual, el número (`vat`) queda vacío, se loguea como nota en el propio contacto hijo y se @-menciona al vendedor de turno (rotación round-robin vía `res.company.woo_salesperson_ids`).
4. Condición fiscal AFIP: ver sección 10 — nunca se adivina, si no hay mapeo configurado queda sin setear.
5. Si el pedido no trae ningún CUIT/DNI, no se crea nada — se factura directo al cliente padre (comportamiento por defecto de Odoo).

**Aislado de la sincronización de `vat` de Odoo entre padre e hijos** (`res.partner.woo_billing_identity`): por defecto, Odoo trata el campo `vat` como **un solo valor compartido** por todo el árbol padre/hijos de un contacto (para que una dirección secundaria de la misma empresa comparta el mismo CUIT) — eso es exactamente lo contrario de lo que necesita esta función, donde cada hijo de facturación representa una identidad fiscal **distinta** bajo el mismo cliente. Sin este aislamiento, cargar el CUIT del segundo hijo pisaba en silencio el CUIT del primero (y del cliente padre) por la sincronización nativa de Odoo — se detectó durante el backfill retroactivo de pedidos reales del 2026-08-16. Los contactos hijo de facturación quedan marcados (`woo_billing_identity = True`) para que ni empujen su CUIT hacia el padre ni lo reciban de él.

Este contacto hijo es el que termina como **`partner_invoice_id`** en la orden de venta (sección 7) — tanto en la que se genera sola al ganar la oportunidad como en una cotización armada a mano con el botón "Nueva Cotización".

### 6.2 Oportunidad (`crm.lead`)

- Se crea como **`type = "opportunity"`**, nunca `"lead"` — un registro `type=lead` no tiene barra de etapas ni botón "Ganado" visibles en el formulario de Odoo, y esta instancia no tiene activada la función "Leads" del CRM. Si se creara como lead, quedaría invisible y sin forma de ganarlo.
- Nombre: `WooCommerce Order #<número>`.
- `woo_order_id` queda guardado (índice único) — evita crear una segunda oportunidad si el mismo evento se reprocesa.
- **Etiqueta + Origen** (`res.company.woo_opportunity_tag_name`, agregado 2026-09-01): si está configurada, se aplica como etiqueta de la oportunidad (`tag_ids`, `crm.tag`) **y** se busca/crea el mismo texto como `source_id` (`utm.source`) — mismo texto para ambos campos, nunca solo uno. Vacío = no toca ninguno de los dos.
- **Idioma** (`lang_id`): mismo criterio que en el contacto (párrafo de 6.1) — `es_AR` si está instalado, nunca forzado.

### 6.2bis Asignación de vendedor y actividad de seguimiento

Agregado 2026-08-31, prioridad corregida 2026-09-02, aplica a toda oportunidad creada desde WooCommerce o desde un formulario web (sección 4bis). Una única prioridad para **ambas** cosas (antes de la corrección, `user_id` siempre era el vendedor de turno sin importar el Referente -- confirmado con el usuario que estaba mal, el Referente debe ser también el dueño, no solo recibir la actividad):

1. **Referente** del producto de interés (`product.template.referente_id`), si tiene uno: se convierte en el `user_id` de la oportunidad **y** recibe la actividad. En este caso el pool de round-robin ni se consulta (no avanza su puntero de rotación).
2. Si no hay Referente: el que corresponda por turno del pool `res.company.woo_salesperson_ids` (`_woo_next_salesperson`, sección 3.3 del manual de usuario) — se convierte en `user_id` **y** recibe la actividad. Asignación **incondicional**, no depende de si el campo ya tenía algo cargado (corregido 2026-09-02: `crm.lead.user_id` trae un default nativo de Odoo, `env.user` = el usuario técnico que procesa el evento, que se aplica siempre al crear -- una condición "solo si estaba vacío" nunca se cumplía en la práctica).
3. Sin Referente y sin nadie en el pool: no se crea ninguna actividad, y `user_id` queda con el default nativo de Odoo (el usuario técnico, nunca vacío).

**Oportunidad creada a mano** (no por estos 2 caminos): recibe actividad también, pero nunca toca `user_id` ni consulta el pool de round-robin — Referente si el producto tiene uno, si no quien la creó (`create_uid`). El pool de round-robin queda reservado para leads sin dueño obvio.

Quien recibe la actividad (Referente, vendedor de turno, o creador) recibe además una notificación de Odoo -- por defecto un email real, salvo que haya cambiado su preferencia de notificación a "En Odoo" (ver manual de usuario, sección 7.1).

---

## 7. Oportunidad "Ganada" → orden de venta

Se dispara **solo** cuando `stage_id` pasa a cualquier etapa con `is_won = True` (nunca una etapa hardcodeada por nombre/id) — tanto al crear la oportunidad ya ganada como al editarla después. Aplica igual a oportunidades creadas a mano, no solo a las que vienen de WooCommerce.

Es idempotente de dos formas: por `opportunity_id` (no crea una segunda orden si ya existe una para esa oportunidad) y por `woo_order_id` (no crea una segunda orden para el mismo pedido de WooCommerce, aunque la primera se haya borrado y la oportunidad se gane de nuevo).

**Si la oportunidad viene de WooCommerce** (tiene `woo_order_id`), la orden se arma releyendo el payload original guardado en `app.event.inbound` (el payload nunca se copia a la oportunidad, se relee cada vez):

- **Líneas de pedido**: producto existente (por `woo_product_id`) o creado al vuelo desde el `line_item` si el sync de catálogo todavía no lo trajo; cantidad y precio del pedido.
- **Moneda**: la del pedido original, por código ISO — se aplica en un `write()` después de crear (si va en el `create()`, el cálculo del pricelist la pisa).
- **Origen** (`sale.order.origin`): siempre el texto `"WooCommerce"`, sin variantes por canal.
- **Referencia del cliente** (`sale.order.client_order_ref`): el sitio que refirió la venta, **solo** si WooCommerce marcó el pedido como "Referido" (meta `_wc_order_attribution_source_type = referral`). Directo/Orgánico/Desconocido no completan este campo.
- **Dirección de facturación** (`sale.order.partner_invoice_id`): el contacto hijo por CUIT de la sección 6.3, si el pedido trae uno. Si no trae CUIT, queda el comportamiento por defecto de Odoo (factura al cliente padre).
- La nota del cliente del checkout (si la dejó) se postea en el chatter de la orden.
- **La orden se confirma automáticamente** (pasa a estado `sale`) si tiene líneas reales — esto es justamente lo que dispara la creación automática de proyecto/tarea de `sale_project` para productos con `service_tracking = task_in_project`. Una orden sin líneas (oportunidad armada a mano) queda en borrador.
- Se manda el mail "Hemos recibido tu pedido" (plantilla editable en Ajustes → Técnico → Email → Plantillas), solo si la oportunidad viene de WooCommerce.

**Si la oportunidad NO viene de WooCommerce**: orden en borrador, solo partner + vendedor, sin líneas, sin confirmar — comportamiento nativo de Odoo, sin ninguna automatización de por medio.

### 7.1 Botón nativo "Nueva Cotización"

El botón de Odoo para armar una cotización adicional a mano sobre una oportunidad (gane ya tenga orden o no) **también** trae las líneas de producto/origen/referencia/dirección de facturación si la oportunidad es de WooCommerce, con el mismo código que arma la orden automática. A propósito **no** trae `woo_order_id`: ese campo es único y solo lo lleva la orden "canónica" generada automáticamente — una segunda cotización manual no debe chocar contra esa restricción.

---

## 8. Pedido cancelado en WooCommerce

- **Detección**: cron "WooCommerce: check cancelled orders" cada 1 hora (+ botón manual "Check Cancelled Orders" en la conexión) — pide a la API `status=cancelled` desde la última revisión (`last_cancelled_check`).
- **Apagado por defecto.** Interruptor "Cancelar pedidos automáticamente" en el menú de la integración → **Configuración** (no en Ajustes generales del sistema — confirmado con el usuario 2026-08-16). Es un ajuste por empresa (`res.company.woo_auto_cancel_orders`), no por conexión: si hay varias conexiones para la misma empresa, cambiarlo afecta a todas.
- Con el interruptor **activo**:
  - Si existe una orden de venta para ese pedido y no tiene ninguna factura activa (una factura ya cancelada no cuenta) → se cancela (`action_cancel()`) y se deja una nota en el chatter.
  - Si tiene una factura activa → **no se cancela**, se posta una nota de alerta en el chatter para que alguien lo revise a mano.
  - Si todavía no hay orden de venta (la oportunidad no se ganó) → no hace nada.
- Con el interruptor **apagado**: el evento igual se guarda en el buffer (queda en estado `done`, sin ninguna acción) — así el dato no se pierde si más adelante se activa el interruptor.

---

## 9. Catálogo de productos

- Cron diario "WooCommerce: sync product catalog" (+ botón "Update Products" en la conexión).
- **Producto nuevo**: se crea con `type=service`, `service_tracking=task_in_project` (proyecto+tarea automáticos al confirmar una orden), y si la empresa está registrada ante AFIP como "IVA Sujeto Exento", con el impuesto "IVA Exen" en ventas y compras.
- **Producto ya existente** (matcheado por `woo_product_id`): **solo se actualiza el precio**. Nombre, SKU y cualquier otro campo nunca se pisan, para no perder una edición manual hecha en Odoo.

---

## 10. Condición fiscal (AFIP)

Solo mapeo **exacto** por `woo.fiscal_condition.mapping` (valor de WooCommerce normalizado en minúsculas, sin espacios de más, contra `woo_value`). Si existe una fila para ese valor → se usa su `responsibility_type_id`.

**Esta condición fiscal también decide el tipo de identificación** (agregado 2026-08-31, `_woo_resolve_identification_type_xmlid`) del contacto hijo de facturación (sección 6.3), con prioridad sobre el conteo de dígitos que se usaba antes como única señal:

| Condición fiscal resuelta | Tipo de identificación |
|---|---|
| Cliente del Exterior (`l10n_ar.res_EXT`) | ID extranjero (`l10n_latam_base.it_fid`) |
| Consumidor Final (`l10n_ar.res_CF`) | DNI (`l10n_ar.it_dni`) |
| IVA Responsable Inscripto (`l10n_ar.res_IVARI`) o Responsable Monotributo (`l10n_ar.res_RM`) | CUIT (`l10n_ar.it_cuit`) |
| Cualquier otra condición fiscal, o sin condición fiscal resuelta | Vuelve al conteo de dígitos (sección 6.3, punto 3) como respaldo |

**Si no existe mapeo, no se adivina nada.** Antes había un fallback que buscaba el nombre AFIP más parecido (stdlib `difflib`) y guardaba esa adivinanza como mapping nuevo — se sacó, porque adivinaba mal (ej: "monotributista" terminaba mapeado a "Monotributista Social" en vez de "Responsable Monotributo", pisando el mapeo real que el usuario ya tenía cargado). Ahora, sin mapeo exacto, la condición fiscal AFIP del contacto hijo queda **sin setear** y se loguea un warning (`_logger.warning`) con el pedido y el valor sin mapear, para que alguien cargue la fila que falta en Ajustes → Mapeo de condición fiscal.

---

## 10.5 Formularios web (Bit Integrations) → contacto + oportunidad

Camino paralelo al de WooCommerce (sección 6), para los formularios de la página web. Cada formulario tiene su propia URL fija (`/api/v1/lead/<identificador>`) y su propia fila de configuración en **Mapeo de Formularios Web** (`webhook.form.mapping.line`). A 2026-09-12 existen 3: **Lead Magnet**, **Consulta**, y **Masterclass** (agregado esa fecha, un formulario de inscripción a eventos tipo Masterclass, reutilizando toda esta misma infraestructura).

- Sin firma HMAC todavía (a diferencia de los webhooks de WooCommerce, sección 3) — Bit Integrations no tiene un mecanismo de firma confirmado aún, el controller acepta cualquier POST por ahora.
- Idempotencia por `bit_form_external_id`, no por contenido: cada request recibe un id aleatorio propio (no un hash del cuerpo), porque dos envíos genuinamente distintos con el mismo texto (la misma persona mandando "consulta" dos veces) antes se colapsaban en un solo evento. El contacto sigue deduplicándose por email/teléfono igual que siempre — lo que nunca se dedupe es la oportunidad.
- Un envío sin nombre, sin email y sin teléfono (típico del botón "enviar solicitud de prueba" de Bit Integrations) no crea nada — se trata como una prueba de conectividad.
- Nombre de la oportunidad: `(identificador del formulario) Nombre` — corregido 2026-09-12 (antes tenía dos puntos, `(identificador): Nombre`, que combinado con el asunto nativo de la notificación de actividad de Odoo generaba un asunto de mail duplicado/confuso).
- **Producto/evento vinculado** (`product_interest_id`, agregado 2026-09-12 para Masterclass): cada fila de mapeo puede tener un Producto/Evento fijo -- para un formulario de inscripción a un evento puntual (donde el evento no viaja en el cuerpo del formulario), la oportunidad queda vinculada a ese producto con certeza, sin adivinar por texto libre (a diferencia de "categoria"/"producto" del formulario de Consulta, que sí se resuelven por texto).
- **Datos sin campo propio en el CRM** (Profesión y Pregunta del formulario de Masterclass, agregado 2026-09-12): quedan como nota en el chatter de la oportunidad (`_bit_apply_extra_notes`), nunca se pierden aunque no tengan una columna dedicada -- mismo patrón que ya usaba categoria/producto de Consulta.
- Asignación de vendedor + actividad de seguimiento: exactamente la misma regla que la sección 6.2bis (Referente del producto, si no turno rotativo, si no vacío) -- se aplica igual sin importar si la oportunidad vino de WooCommerce o de un formulario web.

**Bug real corregido 2026-09-12 (escapado de `<br/>` en notas de chatter):** `message_post(body="<br/>".join(...))` con un `str` común hace que Odoo escape también el propio separador `<br/>` (se veía literal en el chatter en vez de un salto de línea real) -- afecta tanto a Consulta (categoria/producto) como a las notas nuevas de Masterclass. Corregido uniendo con `markupsafe.Markup("<br/>")` en vez de un `str.join()` común; cada valor libre sigue yendo por `html.escape()` antes de unirse, así que el fix no reabre ningún riesgo de inyección.

---

## 11. Permisos

- Grupo **Woo Técnico** (`integrator_events.group_woo_technical`): el único permiso que otorga es lectura de Eventos (`app.event.inbound`). Se asigna a mano por usuario en Ajustes → Usuarios → Permisos. No da acceso a nada más.
- Los Administradores del sistema (`base.group_system`) siempre tienen acceso completo a Eventos — no necesitan este grupo.
- Antes cualquier usuario interno de Odoo podía leer Eventos (`base.group_user`); eso se sacó — ahora hace falta el grupo Woo Técnico o ser Administrador.
- El botón "Eventos" en la conexión, y el menú correspondiente, se ocultan solos para quien no tenga ese permiso.

---

## 12. Deuda conocida / lo que falta

- `order.updated` no tiene handler implementado todavía — cualquier evento con ese topic queda en estado `error` con el mensaje "No handler registered".
- El entorno de desarrollo usa un túnel de ngrok (URL que cambia cada vez que se reinicia) como URL pública del webhook — ver `README.md` sección 3 para cómo se configura contra una tienda real con HTTPS público.
