# Plantilla de reporte de estimación de costes

Estructura recomendada, tanto para un artifact/mensaje en chat como para un `.md` versionado en el repo (mismo patrón que `estimacion-hub-spoke-vpn.md` en la raíz).

## 1. Cabecera

- Nombre del entorno/arquitectura.
- Región de referencia, moneda, tipo de cambio si aplica (y aviso de que es orientativo).
- Convención de cálculo (730 horas/mes es el estándar de Azure).
- Fecha de la consulta de precios (los precios cambian; una estimación de hace 3 meses no es fiable).

## 2. Totales primero (si el formato lo permite)

Antes del detalle línea a línea, un resumen con el/los total(es) mensual(es) — y anual si aporta valor — de cada escenario. Si hay rango por partidas variables (tráfico, logs), mostrar el rango, no solo el punto medio.

## 3. Desglose por categoría

Agrupar líneas por categoría funcional, no por orden alfabético de recurso — facilita que el lector entienda qué está pagando y por qué:

- Cómputo
- Almacenamiento persistente / disks
- Base de datos
- Almacenamiento de objetos (si aplica)
- Red y exposición (separar claramente **egress del cluster** de **ingress/exposición pública** si ambos existen — son recursos distintos)
- Seguridad (Key Vault, WAF, etc.)
- Registro de contenedores / CI-CD (si aplica)
- Observabilidad

Cada línea: recurso, SKU/tier/tamaño usado, precio mensual. Si el número depende de una asunción, márcalo visualmente (icono, superíndice, nota al pie) — nunca lo dejes indistinguible de un número confirmado.

Si hay más de un escenario razonable (p. ej. sizing recomendado vs. ajustado por presupuesto), usar columnas paralelas en vez de duplicar toda la tabla.

## 4. Total

Suma por categoría + total general, coherente con el resumen de la sección 2.

## 5. Supuestos y puntos abiertos

Lista explícita de todo lo que no estaba fijado en los requisitos originales y que se tuvo que asumir para poder calcular un número. Para cada punto: qué se asumió, por qué importa (cuánto puede mover el total si la asunción es incorrecta), y qué decisión falta confirmar.

Esta sección es la más importante del reporte — es lo que evita que una estimación se confunda con una cotización cerrada.

## 6. Metodología / fuentes

- Qué API o fuente se usó (Azure Retail Prices API, con URL).
- Filtros aplicados (región, tipo de precio).
- Recordatorio de que esto **no es una cotización oficial** y que la Azure Pricing Calculator oficial debe usarse para confirmar antes de aprobar presupuesto.
- Enlaces a las páginas de pricing oficiales de los servicios principales, si se quiere dar trazabilidad adicional.

## 7. (Opcional) Recursos añadidos a posteriori / optimizaciones de coste

Si tiene valor para el lector, una sección aparte con:
- Recursos que no son necesarios desde el día 1 pero aparecen frecuentemente en la evolución natural de la arquitectura (WAF, CDN, caché, mensajería, etc.), con coste estimado orientativo.
- Optimizaciones de coste aplicables (Reserved Instances, Savings Plans, Azure Hybrid Benefit, pausar capacidad fuera de horario, etc.) con el % de ahorro esperado.
