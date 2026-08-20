---
name: azure-cost-estimate
description: "Build Azure cost estimates from LIVE Azure Retail Prices API data (prices.azure.com), not memorized/static pricing. Produces category-grouped monthly/annual cost breakdowns with explicit assumptions and open questions flagged, in the style equivalent to the Azure Pricing Calculator. WHEN: \"estimar costos\", \"estimación de costos\", \"cost estimate\", \"cuánto cuesta\", \"presupuesto Azure\", \"Azure pricing\", \"calculadora de Azure\", \"Azure Pricing Calculator\", \"cost breakdown\", \"budget for Azure\", \"how much would this cost on Azure\", \"desglose de costes\"."
license: MIT
metadata:
  author: local
  version: "0.1.0"
---

# Azure Cost Estimate

> Origen: metodología desarrollada durante una estimación real para un entorno DEMO de Liferay DXP + Elasticsearch sobre AKS (agosto 2026). Se documenta aquí como skill reutilizable para que futuras estimaciones en este repo sigan el mismo estándar en vez de partir de precios recordados/desactualizados.

## Cuándo usar esta skill

Actívala cuando el usuario pida presupuestar, estimar costes, comparar escenarios de sizing, o preguntar "¿cuánto costaría esto en Azure?" para cualquier arquitectura — greenfield, migración, o cambio de sizing sobre algo ya desplegado.

No sustituye a `azure-prepare` (que genera IaC) ni a las skills de migración/diagnóstico — se puede usar antes de `azure-prepare` (para presupuestar antes de construir) o de forma independiente.

## Principio central

**Nunca calcules precios de memoria.** Los precios de Azure cambian con frecuencia y varían por región, tier y tipo de oferta. Consulta siempre la **Azure Retail Prices API** en vivo — es pública, no requiere autenticación y es la misma fuente de datos que alimenta la Azure Pricing Calculator oficial.

```
GET https://prices.azure.com/api/retail/prices?api-version=2023-01-01-preview&$filter=<OData filter>
```

Ver [references/retail-prices-api.md](references/retail-prices-api.md) para sintaxis de filtro, valores de `serviceName`/`productName` habituales por categoría de recurso, y gotchas encontrados en la práctica (Load Balancer vive en la región "Global", los Managed Disks facturan por tier fijo, etc.).

## Workflow

1. **Reunir sizing/requisitos** — qué recursos, qué SKUs candidatos, qué tamaños/tiers, para qué entorno (demo/prod), y con qué nivel de HA.
2. **Confirmar los parámetros que más impactan el coste ANTES de calcular** — normalmente: región, moneda, número de nodos/instancias, y cualquier decisión con delta de coste grande (p. ej. WAF sí/no, tier de un servicio). Usa preguntas cortas y accionables (`AskUserQuestion`), máximo 3-4 a la vez. No preguntes por parámetros que no cambian el resultado de forma material.
3. **Consultar la Retail Prices API** por cada recurso, filtrando por región y SKU/meter. Descarta filas de tipo `Reservation`/`Spot` salvo que se pida explícitamente. Guarda las respuestas en el scratchpad y parsea con `python3` (no asumas `jq` disponible).
4. **Construir la tabla de desglose agrupada por categoría** (Cómputo, Almacenamiento, Base de datos, Red, Seguridad, Observabilidad, CI/CD, etc.), con precio mensual (730h/mes es la convención estándar de Azure) y, si aporta valor, anual.
5. **Si hay más de una configuración razonable** (p. ej. sizing recomendado vs. ajustado por presupuesto), presenta los escenarios lado a lado en vez de elegir uno por el usuario.
6. **Marca explícitamente cualquier número que dependa de una asunción no confirmada** (tamaño real de storage, volumen de logs, tier de un servicio, mecanismo de exposición, etc.) en una sección de supuestos/puntos abiertos — nunca los mezcles silenciosamente en el total sin flag.
7. **Cierra siempre con una nota de metodología**: fuente (Retail Prices API), región, tipo de cambio usado si se convierte desde USD (y que es orientativo, no en tiempo real), fecha de la consulta, y recomendación explícita de validar en la Azure Pricing Calculator oficial antes de aprobar presupuesto — esto no es una cotización oficial.

## Formato de salida

- **Para compartir en el chat**: usa el formato visual (tabla/artifact) que mejor comunique el desglose; ver [references/report-template.md](references/report-template.md) para la estructura de secciones recomendada.
- **Si el usuario pide guardarlo como archivo en el repo**: sigue el mismo patrón que `estimacion-hub-spoke-vpn.md` (raíz del repo) — un `.md` autocontenido con la tabla de desglose, escenarios, supuestos y fuentes. No mezcles el artifact HTML de una sesión de chat con el código del repo — son cosas distintas (uno es un entregable de conversación, el otro es contenido versionado).

## Notas aprendidas (gotchas de la API)

Ver [references/retail-prices-api.md](references/retail-prices-api.md) para el detalle completo. Resumen rápido:

- Managed Disks (Premium/Standard SSD v1) facturan por el **tier fijo más cercano por encima** del tamaño solicitado (P4=32GB, P6=64GB, P10=128GB…), no de forma proporcional al GB exacto.
- El meter base de **Load Balancer Standard** vive bajo `armRegionName eq 'Global'`, no bajo la región concreta.
- **Application Gateway v2** = Fixed Cost (por hora, según SKU) + Capacity Units consumidas (variable con tráfico real) + IP pública propia — nunca lo reduzcas a un solo número fijo sin avisar de la parte variable.
- **PostgreSQL Flexible Server**: storage y backup son meters aparte del cómputo; el backup con retención por defecto (7 días) es gratis hasta el 100% del storage provisionado.
- **Azure Monitor / Log Analytics** (ingesta de Container Insights) es casi siempre la partida más variable de cualquier estimación con Kubernetes — preséntala siempre como rango (bajo/medio/alto), nunca como un único número.
