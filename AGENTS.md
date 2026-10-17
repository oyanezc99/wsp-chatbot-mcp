# Instrucciones para agentes de código

1. **La fuente de verdad es `specs/SPEC-2026-10-16-02-mvp-crm-agente-respuesta.md`.**
   - Construye solo lo que figura en su sección 4.
   - Respeta las decisiones D1 a D17 de su sección 9.
   - No construyas nada de su sección 5 (fuera de alcance), aunque aparezca en otros documentos.
2. **Los otros documentos son solo contexto, no requisitos:**
   - `specs/SPEC-2026-10-16-01-…` es la visión del producto. Sus erratas prevalecen sobre su texto.
   - La propuesta del 14-oct-2026, en la raíz del repo.
   - `docs/analisis/`.
3. **Si una tarea pide algo fuera del alcance o contradice una decisión**, detente y pregunta. Un cambio de alcance se hace modificando el spec en un PR, no en el código.
4. **Reglas que no se simplifican** (detalle en los anexos A, B, E y F del spec):
   - Validar `X-Hub-Signature-256` sobre el cuerpo crudo.
   - Idempotencia (D9): POST crudo único por `sha256`; mensajes únicos por `wamid`; estados monótonos que guardan `pricing` y `errors` aunque lleguen fuera de orden.
   - Outbox de salida, sin reintentos a ciegas.
   - `organizacion_id` en todas las tablas de negocio, con filtro global y prueba de aislamiento (D14).
   - Un único servicio de envío que verifica la ventana de 24 h, el opt-out (D17) y el interruptor del bot.
   - Un turno a la vez por conversación (lease, no locks consultivos), con agrupación de ráfagas.
   - Validar en el servidor los `faq_ids` y las cifras de cada respuesta.
   - Enmascarar los datos antes del LLM.
   - Nunca guardar secretos en el código ni en la base de datos.
5. **Al terminar una historia:** cumple la definición de terminado del Anexo F y deja las pruebas en verde. Si cambiaste el agente, corre el set de evaluación.
6. **Pruebas sin gastar mensajes:** usa el simulador de webhooks, el `FakeWhatsAppClient` y el motor LLM falso. El número de prueba de Meta se usa solo para la prueba de humo.
