# Checkpoint 4 · Integraciones avanzadas (HubSpot + Gmail + Slack)

**Autor:** Mauricio Rubino
**Entregable:** `checkpoint4_mauricio_rubino.json` (workflow Manager exportado de n8n)
**Proyecto integrador:** agente de calificación de leads de *Horizonte Este Propiedades* (inmobiliaria ficticia de Maldonado y Punta del Este)

---

## Evolución del proyecto

| Módulo | Qué se agregó |
|---|---|
| M1 | Agente base: Gmail Trigger → AI Agent (Tools Agent) + herramienta Gmail → log en Slack |
| M2 | Arquitectura Manager-Worker: Router de triaje con taxonomía cerrada + 2 Workers como sub-workflows |
| M3 | Memoria de largo plazo en Airtable por Session_ID + summarization con modelo económico |
| **M4** | **Integración con CRM (HubSpot), Gmail y Slack con controles preventivos** |

## Herramientas conectadas (OAuth2)

| Rol en el caso | Herramienta | Lectura (pasado) | Escritura (futuro) |
|---|---|---|---|
| CRM / fuente única de verdad | **HubSpot** | Buscar contacto por email | Crear o actualizar el contacto (estado del lead, etapa, nota) |
| Casilla de soporte | **Gmail** | Gmail Trigger (correos no leídos) | Create Draft: borrador de respuesta en el mismo hilo |
| Canal del equipo | **Slack** | — | Mensaje de trazabilidad en `#agente-observabilidad` |

## Los 4 controles que pide la rúbrica

```
[Gmail Trigger: correo entrante]
        │
        ▼
① [IF ¿Es correo automático?] ── Sí ─▶ Stop (corta el bucle de auto-respuestas)
        │ No
        ▼
[Memoria Airtable] → [Router de triaje] → [Worker 1 / Worker 2 (AI Agents)] → [Registro unificado]
        │
        ▼
④ [Set · Payload limpio y validado] → [IF ¿Email válido?]   (evita el error 400)
        │
        ▼
② [HubSpot · Buscar contacto (Look up)] → [IF ¿Existe?] ─ Sí ▶ [Actualizar contacto]
                                                        └ No ▶ [Crear contacto]        (evita el error 409)
        │
        ▼
④ [Set · Payload mínimo para Slack] → [Slack]
        │
        ▼
③ [Gmail · Create Draft]  ← borrador para aprobación humana (Human-in-the-loop)
```

1. **IF anti auto-reply**, inmediatamente después del trigger. Ignora asuntos con *Auto-reply, Automatic reply, Respuesta automática, Out of office, Fuera de la oficina, Undeliverable, Delivery Status Notification, Mail delivery failed*; remitentes *no-reply, noreply, do-not-reply, mailer-daemon, postmaster*; y el header `Auto-Submitted`.
2. **Look up antes del Create**: el contacto se busca por email en HubSpot. Si existe se actualiza; si no, se crea. Nunca se intenta crear un duplicado.
3. **Create Draft**: la respuesta al cliente (y el acuse de recibo en escalamientos) se guarda como **borrador** en el hilo original. Ningún correo llega al cliente sin aprobación humana.
4. **Set de limpieza**:
   - antes del CRM: solo `email` (normalizado y validado con regex), `firstname`, `lastname`, `estado`, `lead_status`, `lifecycle` y una `nota` de hasta 500 caracteres;
   - antes de Slack: solo campos de negocio (sin JSON de payloads, HTML ni binarios).

## Mínimo privilegio

- **HubSpot:** el flujo solo usa *contactos* (buscar, crear y actualizar). Scopes necesarios: `crm.objects.contacts.read` y `crm.objects.contacts.write`.
- **Gmail:** lee la casilla de consultas y crea borradores. Al cliente nunca se le envía un correo directo; los únicos envíos son notificaciones internas (equipo comercial, tasaciones y supervisor).
- **Slack:** publica en un único canal (`chat:write`).
- **Airtable (memoria, M3):** token personal con acceso solo a la base de memoria y scopes `data.records:read`, `data.records:write`, `schema.bases:read`.

## Pruebas de regresión realizadas

| Prueba | Resultado |
|---|---|
| Lead nuevo (compra de apartamento) | Router → Worker 1 (lead CALIENTE) → contacto creado en HubSpot → ficha derivada al equipo comercial → respuesta guardada como **borrador** en el hilo |
| Mismo cliente otra vez | El Look up encuentra el contacto → **se actualiza sin duplicar** |
| Respuesta automática ("Fuera de la oficina") | El IF ① la detecta y el flujo termina en **Stop** sin ejecutar IA, CRM, Slack ni Gmail |
| Escalamiento / intención dudosa | Alerta interna al supervisor y acuse de recibo al cliente como **borrador** |

## Cómo importarlo

1. En n8n: **Workflows → Import from File** → `checkpoint4_mauricio_rubino.json`.
2. Configurar las credenciales: Gmail OAuth2, HubSpot OAuth2, Slack OAuth2, Anthropic y Airtable.
3. En los nodos *Delegar a Worker 1 / Worker 2*, elegir los sub-workflows de los Workers (los Workers del M3 se usan sin cambios).
4. Ejecutar con **Fetch Test Event** en el trigger → **Execute workflow**.
