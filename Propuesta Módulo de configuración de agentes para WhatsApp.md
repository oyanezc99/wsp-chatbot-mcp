# Propuesta: Módulo de configuración de agentes para WhatsApp

Oct 14, 2026 · @Omar Jesus Yañez Carlos

## Resumen ejecutivo

Se propone una solución en dos piezas: un **módulo interno** con el que nuestro equipo genera agentes de IA para cada negocio cliente y los conecta a WhatsApp usando el MCP oficial de Meta (WhatsApp Business Tools MCP), y un **runtime** que atiende en vivo los chats de los clientes finales de ese negocio.

El negocio cliente no usa el módulo: recibe un agente ya configurado y funcionando en su número de WhatsApp. El MCP se usa solo durante el alta (cuenta, número, plantillas, webhook y mensaje de prueba), porque Meta lo define como una herramienta para desarrollo y pruebas, no para envíos en producción a escala. En la operación diaria, los mensajes entran por el webhook al runtime y las respuestas salen por la WhatsApp Cloud API.

## Objetivos y alcance

El objetivo es que nuestro equipo pase de “este negocio quiere un agente” a un agente publicado y respondiendo en su WhatsApp en una sola sesión de trabajo, sin escribir código ni configurar Meta a mano.

**Objetivos**

- Generar la configuración completa de un agente a partir de un brief del negocio.
- Dar de alta el número, las plantillas y el webhook de cada cliente mediante el MCP oficial.
- Atender en vivo los chats de los clientes finales desde el runtime, las 24 horas.
- Derivar a un asesor humano del negocio con el contexto completo de la conversación.

| Incluye (v1) | Fuera de alcance (v1) |
| --- | --- |
| Generador de agentes a partir de un brief (instrucciones, tono, límites, FAQ, reglas) | Desarrollar una integración propia con la API de Meta para onboarding |
| Aprovisionamiento vía WhatsApp Business Tools MCP: número, plantillas, webhook, prueba | Llamadas de voz y videollamadas |
| Runtime que recibe el webhook y ejecuta el agente generado | Entrenamiento o ajuste fino de modelos propios |
| Base de conocimiento por agente (FAQ, documentos) | Canales distintos a WhatsApp |
| Revisión, chat de prueba, versiones y publicación | Campañas masivas de marketing |
| Derivación a asesor y panel de métricas básicas | Integraciones a medida con sistemas del cliente (se cotizan aparte) |

## Funcionalidades del módulo

El generador produce un borrador de agente con todos sus bloques; el administrador solo revisa y ajusta antes de publicar. Los dos últimos bloques se ejecutan a través del MCP oficial.

| Bloque | Qué hace | Ejemplo |
| --- | --- | --- |
| Brief del negocio (entrada) | Formulario guiado: rubro, objetivo del agente, productos, horarios, documentos y ejemplos de chats | Tienda de ropa que quiere atender cambios y estado de pedidos |
| Generación del agente | Un LLM convierte el brief en la configuración inicial de todos los bloques siguientes | Borrador listo para revisar en un minuto |
| Identidad y tono | Nombre, saludo, idioma, registro (formal / cercano), firma | “Hola, soy Sofía de Soporte. ¿En qué te ayudo?” |
| Instrucciones y límites | Objetivo, temas permitidos y prohibidos, qué nunca debe prometer | No dar precios que no estén en el catálogo |
| Base de conocimiento | FAQ generadas del brief más PDFs y URLs, indexadas para búsqueda semántica (RAG) | Políticas de cambio y devolución |
| Acciones (herramientas) | Operaciones que el agente puede ejecutar vía API, con permisos por agente | Consultar estado de pedido, agendar cita |
| Reglas de derivación y horarios | Cuándo pasa el chat a un humano, a qué cola y qué decir fuera de horario | Cliente pide asesor o hay 2 respuestas sin resolver |
| Revisión y versiones | Chat de prueba, comparación entre versiones, publicación y vuelta atrás | Probar un cambio de tono antes de publicarlo |
| Plantillas (vía MCP) | Genera las plantillas que el agente necesita y las crea en Meta para aprobación | Recordatorio de cita, seguimiento de caso |
| Conexión WhatsApp (vía MCP) | Revisa el estado de la cuenta, registra el número, apunta el webhook al agente y envía un mensaje de prueba | Número verificado y webhook activo |

Como apoyo transversal, el módulo incluye roles (administrador, supervisor, asesor), registro de auditoría de cada cambio y un panel con conversaciones atendidas, tasa de derivación y temas más consultados.

## Arquitectura propuesta

La arquitectura tiene dos momentos: al crear el agente, el generador arma la configuración y el MCP de Meta prepara la cuenta de WhatsApp; en cada conversación, el webhook entrega el mensaje al agente generado, que responde o deriva.

&#91;embedded content: arquitectura del módulo · 10 componentes\]

Solo nuestro equipo usa la franja superior; el negocio cliente vive únicamente en la inferior. El webhook que el MCP configura en la cuenta de WhatsApp apunta al webhook del runtime. El runtime envía las respuestas del agente y de los asesores con la Cloud API (`POST /{phone_number_id}/messages`), autenticado con un **token de usuario de sistema** del negocio y no con el inicio de sesión del MCP. Meta indica que esta versión del MCP está pensada para desarrollo y pruebas, no para envíos en producción a escala, y que cada cambio de estado exige a una persona autenticada ([Meta for Developers](https://developers.facebook.com/blog/post/2026/09/15/meta-business-messaging-mcp-ai-agent/)).

## Modelo de datos principal

El centro del modelo es la versión del agente: las conversaciones guardan con qué versión se atendieron, así un cambio nunca altera el historial ni impide volver atrás.

| Entidad | Campos clave | Relación |
| --- | --- | --- |
| Organizacion | id, nombre, zona horaria, plan | Tiene muchos números y agentes |
| BriefAgente | id, rubro, objetivo, productos, horarios, documentos, ejemplos de chats, creado\_por | Entrada del generador; origina una o más versiones |
| NumeroWhatsApp | id, phone\_number\_id, waba\_id, estado de registro, agente\_principal\_id | Registrado y verificado vía MCP |
| Agente | id, nombre, estado (borrador / activo / pausado), version\_publicada\_id | Tiene muchas versiones |
| VersionAgente | id, numero, brief\_id, instrucciones, tono, modelo, temperatura, max\_tokens, origen (generada / editada), fecha | Snapshot inmutable de la configuración |
| PlantillaWhatsApp | id, version\_id, nombre, categoría, idioma, estado de aprobación en Meta | Creada en Meta vía MCP |
| FuenteConocimiento | id, agente\_id, tipo (FAQ / PDF / URL), estado de indexación | Sus fragmentos van al índice vectorial |
| Herramienta | id, nombre, endpoint, esquema de parámetros, requiere\_confirmacion | Se habilita por versión de agente |
| ReglaDerivacion | id, version\_id, condición, cola\_destino, prioridad | Se evalúa en cada mensaje |
| Conversacion | id, numero\_id, cliente\_wa\_id, estado (bot / humano / cerrada), version\_id, asesor\_id | Tiene muchos mensajes |
| Mensaje | id, conversacion\_id, emisor (cliente / agente / asesor), contenido, wamid, fecha | Trazabilidad completa |
| OperacionMCP | id, herramienta MCP usada, parámetros, resultado, usuario autenticado, fecha | Registro de cada cambio hecho en Meta |
| Auditoria | id, usuario, entidad, acción, antes / después, fecha | Registra cada cambio de configuración |

Se sugiere PostgreSQL con la extensión pgvector para no operar una base vectorial aparte en la v1.

## Flujo de atención de un chat

Cada mensaje pasa por tres filtros antes de que el agente conteste: si ya lo atiende un asesor, si se cumple una regla de derivación y si la respuesta tiene sustento en la base de conocimiento o en una acción.

&#91;embedded content: flujo de atención · 3 decisiones\]

Si la derivación ocurre fuera del horario humano, el agente envía el mensaje de fuera de horario y el caso queda en cola para el siguiente turno.

### Ejemplo: tienda de ropa

1. Una clienta escribe al WhatsApp de la tienda: “¿Tienen el polo negro en talla M?”.
2. Meta recibe el mensaje y lo envía al webhook del runtime, que el MCP dejó configurado durante el alta.
3. El runtime carga la versión publicada del agente de la tienda y le pasa el mensaje.
4. El agente usa su herramienta de stock, confirma que hay unidades y arma la respuesta con el precio del catálogo.
5. El runtime responde por la Cloud API: “Sí, tenemos el polo negro en M a S/ 49. ¿Te lo separo?”.
6. Si la clienta pide hablar con alguien o el agente no encuentra la respuesta, el chat pasa a la bandeja de un asesor de la tienda.

Si la clienta no escribe en 24 horas, la tienda solo puede contactarla con una plantilla aprobada (por ejemplo, “Tu pedido fue enviado”), creada antes con el MCP.

## Seguridad, políticas de Meta y protección de datos

El agente debe estar acotado al negocio: desde el 15 de enero de 2026 Meta prohíbe en la API de WhatsApp Business los chatbots de IA de propósito general, pero permite los bots de atención, ventas y notificaciones de una empresa ([respond.io](https://respond.io/blog/whatsapp-chatbot-policy-2026), [Gulf News](https://gulfnews.com/technology/no-more-chatgpt-and-perplexity-on-whatsapp-meta-bans-major-ai-chatbots-from-2026-1.500316016)). Por eso el bloque de instrucciones y límites es obligatorio y el agente rechaza temas ajenos al negocio.

- **Acceso al MCP:** se conecta con la cuenta Meta de un administrador y solo a los negocios que este elija; cada cambio de estado requiere a esa persona autenticada y Meta registra cada invocación ([The New Stack](https://thenewstack.io/meta-mcp-whatsapp-business-claude/)). El módulo pide confirmación antes de cada operación MCP y la guarda en OperacionMCP.
- **Ventana de 24 horas:** fuera de ella solo se puede escribir con plantillas aprobadas; el runtime bloquea respuestas libres cuando la ventana cerró.
- **Consentimiento:** se registra el opt-in del cliente y se respeta la baja (“no quiero más mensajes”).
- **Datos personales:** cumplimiento de la Ley N.° 29733 de Protección de Datos Personales del Perú; política de retención configurable y enmascarado de DNI, tarjetas y teléfonos antes de enviar texto al LLM.
- **Secretos:** token de usuario de sistema de cada negocio y llaves del LLM en un gestor de secretos, nunca en la base de datos en texto plano.
- **Webhooks:** validación de la firma de Meta en cada evento entrante.
- **Inyección de instrucciones:** el brief, los documentos y los mensajes del cliente se tratan como datos; el generador no puede quitar los límites obligatorios y las acciones sensibles piden confirmación.
- **Acceso al módulo:** roles por organización, registro de auditoría y doble factor para administradores.

## Plan de implementación por fases

Se propone entregar en cuatro fases, cada una usable por sí sola; la duración de cada fase queda por estimar con el equipo.

| Fase | Entregables | Criterio para pasar a la siguiente |
| --- | --- | --- |
| 1. Conexión vía MCP | Acceso al WhatsApp Business Tools MCP, número registrado, webhook apuntando al runtime, mensaje de prueba | Un mensaje real entra al runtime y recibe respuesta de eco |
| 2. Generador de agentes | Formulario de brief, generación de la configuración, revisión, chat de prueba, versiones | Nuestro equipo genera y publica el agente de un negocio en una sola sesión |
| 3. Conocimiento, plantillas y derivación | RAG, plantillas creadas vía MCP, reglas de derivación, bandeja de asesores, horarios | El agente responde FAQ con fuente y deriva con contexto |
| 4. Acciones y métricas | Herramientas vía API, panel de métricas, auditoría y registro de operaciones MCP | Piloto con clientes reales durante dos semanas sin incidentes graves |

## Riesgos y mitigaciones

El riesgo principal es que el agente invente información; se mitiga obligándolo a responder solo con su base de conocimiento y a derivar cuando no encuentra respuesta.

| Riesgo | Impacto | Mitigación |
| --- | --- | --- |
| Respuestas inventadas (alucinaciones) | Alto | RAG con fuente, umbral de confianza y derivación a humano |
| El generador crea un agente mal acotado | Alto | Límites obligatorios que no se pueden quitar, revisión humana antes de publicar |
| MCP en beta y con despliegue gradual: puede no estar disponible o cambiar | Medio | Confirmar acceso en la fase 1; la configuración manual en Meta queda como plan B |
| Acciones del MCP requieren una persona autenticada | Medio | Tratar la conexión como paso asistido por un administrador, no como proceso desatendido |
| Suspensión del número por Meta | Alto | Agente acotado al negocio, opt-in, calidad de plantillas monitoreada |
| Costo de LLM y mensajes fuera de control | Medio | Presupuesto mensual por agente, modelo económico para FAQ, alertas |
| Fuga de datos personales | Alto | Enmascarado antes del LLM, retención limitada, auditoría |

## Próximos pasos y decisiones abiertas

- [ ] Confirmar que la cuenta de Meta del equipo ya tiene acceso al WhatsApp Business Tools MCP
- [ ] Definir el negocio piloto y los casos de uso del primer agente (soporte, ventas, citas)
- [ ] Definir los campos del brief y un ejemplo completo para probar el generador
- [ ] Elegir proveedor y modelo de LLM para generar y para atender, y fijar presupuesto mensual
- [ ] Confirmar el stack (propuesto: ASP.NET Core + PostgreSQL/pgvector + Angular o React)
- [ ] Revisar la tarifa vigente de mensajes de Meta para Perú
- [ ] Estimar la duración de cada fase con el equipo

## Fuentes

- [Meta lets Claude and Codex configure WhatsApp Business via MCP — The New Stack (15 set 2026)](https://thenewstack.io/meta-mcp-whatsapp-business-claude/)
- [Anuncio de Meta del WhatsApp Business Tools MCP](https://developers.facebook.com/blog/post/2026/09/15/meta-business-messaging-mcp-ai-agent/)
- [Not All Chatbots Are Banned: WhatsApp’s 2026 AI Policy Explained — respond.io](https://respond.io/blog/whatsapp-chatbot-policy-2026)
- [Meta bans major AI chatbots from 2026 — Gulf News](https://gulfnews.com/technology/no-more-chatgpt-and-perplexity-on-whatsapp-meta-bans-major-ai-chatbots-from-2026-1.500316016)
