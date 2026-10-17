# wsp-chatbot-mcp

Plataforma de **CRM + agente IA de respuesta** sobre la API oficial de WhatsApp (Cloud API) para negocios en Perú. El agente atiende las consultas frecuentes las 24 h y deriva a un asesor. Los asesores del negocio atienden y hacen seguimiento a sus clientes desde una bandeja web con CRM.

**Estado:** especificación. El MVP está en `refinado`, pendiente de que lo aprueben Manuel y Omar. Todavía no hay código.

## Por dónde empezar

| Si quieres… | Lee |
| --- | --- |
| Saber **qué se construye** (el contrato del MVP) | [`specs/SPEC-2026-10-16-02-mvp-crm-agente-respuesta.md`](specs/SPEC-2026-10-16-02-mvp-crm-agente-respuesta.md) |
| Ver la visión completa del producto (12 módulos, 6 fases) | [`specs/SPEC-2026-10-16-01-crm-agentes-ia-whatsapp.md`](specs/SPEC-2026-10-16-01-crm-agentes-ia-whatsapp.md) (revisa sus erratas) |
| Leer la propuesta original (14-oct-2026) | [`Propuesta Módulo de configuración de agentes para WhatsApp.md`](Propuesta%20M%C3%B3dulo%20de%20configuraci%C3%B3n%20de%20agentes%20para%20WhatsApp.md) |
| Ver la verificación de datos de Meta y la competencia | [`docs/analisis/revision-2026-10-16.md`](docs/analisis/revision-2026-10-16.md) |

## Cómo se trabaja

- **Manda el spec del MVP.** Lo que no figura en su sección 4 no se construye. Para cambiar el alcance, abre un PR que modifique el spec.
- **Specs** van en `specs/`, con el formato `SPEC-YYYY-MM-DD-NN-slug.md`, y se mantienen idénticos a la copia central de SDD-Specs.
- **Decisiones técnicas** van en `docs/adr/` (ADR de media página). **Runbooks** en `docs/runbooks/`. **Datos del piloto** en `docs/piloto/`.
- La estructura de código sugerida está en el Anexo F del spec del MVP.
