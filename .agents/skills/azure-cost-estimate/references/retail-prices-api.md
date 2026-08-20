# Azure Retail Prices API — guía práctica

Fuente pública, sin autenticación, usada por la propia Azure Pricing Calculator:

```
https://prices.azure.com/api/retail/prices?api-version=2023-01-01-preview
```

## Sintaxis de filtro (OData, parámetro `$filter`)

Usar `curl -G --data-urlencode "\$filter=..."` para evitar problemas de escapado de comillas y espacios. Ejemplo real:

```bash
curl -s -G "https://prices.azure.com/api/retail/prices?api-version=2023-01-01-preview" \
  --data-urlencode "\$filter=armRegionName eq 'westeurope' and armSkuName eq 'Standard_D8s_v5' and priceType eq 'Consumption'"
```

Operadores útiles:
- `eq` — igualdad exacta (`armRegionName eq 'westeurope'`)
- `contains(campo,'texto')` — búsqueda parcial (`contains(productName,'Flexible Server')`)
- `and` para combinar filtros — cuantos más filtros, menos ruido en la respuesta

Campos más usados para filtrar: `armRegionName`, `armSkuName`, `serviceName`, `productName`, `skuName`, `meterName`, `priceType` (`Consumption` = PAYG, `Reservation` = RI, evitar salvo que se pida).

El JSON no viene ordenado ni fácil de leer a ojo — parsear siempre con `python3` (no asumir `jq` instalado):

```python
import json
data = json.load(open("respuesta.json"))
for it in data["Items"]:
    print(it["skuName"], it["meterName"], it["retailPrice"], it["unitOfMeasure"])
```

## `serviceName` / `productName` habituales por categoría de recurso

| Recurso | `serviceName` | Notas |
|---|---|---|
| VM (nodos AKS, VMs sueltas) | `Virtual Machines` | Filtra por `armSkuName`; descarta filas con `Windows` en `productName` si es Linux; descarta `Spot`/`Low Priority` salvo que se pida explícitamente |
| Managed Disks | `Storage`, `productName` contiene `Premium SSD Managed Disks` / `Standard SSD Managed Disks` | Ver gotcha de tiers fijos abajo |
| PostgreSQL Flexible Server | `Azure Database for PostgreSQL`, `productName` contiene `Flexible Server` | Cómputo, storage y backup son **meters distintos** — no asumas que uno incluye el otro |
| Blob Storage | `Storage`, `productName` = `Blob Storage` | Filtrar por `skuName` (`Hot LRS`, `Cool LRS`, etc.) |
| Key Vault | `Key Vault` | Coste por operaciones (10K ops), normalmente negligible salvo alto volumen |
| Container Registry (ACR) | `Container Registry` | `Basic`/`Standard`/`Premium Registry Unit` = coste fijo diario; `Data Stored` = GB/mes aparte |
| Load Balancer | `Load Balancer` | **Ver gotcha de región `Global` abajo** |
| Public IP | Buscar `contains(meterName,'IP Address')` dentro de servicios de red, o `productName` = `Public IP Prefix` | SKU `Standard` para IP estática de producción |
| Application Gateway | `Application Gateway` | Ver gotcha de Fixed Cost + Capacity Units abajo |
| AKS control plane | `contains(serviceName,'Kubernetes')` | `Standard Uptime SLA` = tier Standard con SLA; Free tier = $0, no aparece como meter facturable |
| Azure Monitor / Log Analytics | `Log Analytics` (ingesta/retención) o `Azure Monitor` (otros) | Buscar meter `Data Ingestion` para PAYG; primeros ~5GB/día suelen ser gratis según el workspace |

## Gotchas encontrados en la práctica

### 1. Managed Disks facturan por tier fijo, no por GB exacto
Premium SSD v1 y Standard SSD v1 solo existen en tamaños fijos (P1, P2, P3, P4=32GB, P6=64GB, P10=128GB, P15=256GB…). Si el usuario pide una PVC de 20GB, Azure la factura como el tier igual o superior más próximo (P4=32GB), **no** como 20GB proporcional. Solo Premium SSD v2 factura por GiB real. Documentar siempre qué tier real se está pagando, no el tamaño solicitado.

### 2. Load Balancer Standard vive en `armRegionName eq 'Global'`
El meter base (`Standard Included LB Rules and Outbound Rules`, coste fijo por hora que incluye las primeras 5 reglas) no aparece si filtras por una región concreta — hay que consultar sin filtro de región, o filtrando `armRegionName eq 'Global'` explícitamente. El resto de meters (Data Processed) sí son por región.

### 3. Application Gateway v2 tiene 3 componentes de coste, no 1
- **Fixed Cost** (`Standard Fixed Cost` / `Standard Fixed Cost` en el producto `... WAF v2`) — coste fijo por hora según SKU (`Standard_v2` o `WAF_v2`).
- **Capacity Units** — variable, depende del tráfico real (compute units, throughput, conexiones persistentes — Azure cobra por el máximo de los tres). Para una primera estimación sin datos de tráfico reales, usar 1 CU como suelo y avisar de que puede escalar.
- **IP pública propia** — el Application Gateway necesita su propia IP Standard, aparte de cualquier IP que ya use el cluster para egress.

### 4. AKS crea un Load Balancer + IP pública por defecto para el egress de los nodos
Esto ocurre **aunque no se exponga ningún servicio** — es el mecanismo de salida a internet (SNAT) por defecto (`outboundType: loadBalancer`). Si además se añade un Application Gateway para el ingress, ese LB de egress **no desaparece** — son dos recursos de red distintos con propósitos distintos. Solo desaparece si se cambia el `outboundType` a NAT Gateway o UDR.

### 5. PostgreSQL Flexible Server: backup por defecto es gratis
La retención de backup por defecto (7 días) está incluida sin coste adicional hasta el 100% del storage provisionado. Solo cobra aparte si se supera ese umbral o se activa Long-Term Retention.

### 6. AKS Free tier no tiene meter facturable
El control plane en Free tier no genera ningún meter — es $0 explícito, no "no encontrado". Solo el tier Standard (`Standard Uptime SLA`, ~$0.10/hora) aparece como línea facturable.

### 7. Azure Monitor / Log Analytics es la partida más impredecible en clusters K8s
El volumen de ingesta depende enormemente de la verbosidad de logs, el número de pods, y si se usan Data Collection Rules para filtrar. Nunca lo reduzcas a un único número sin dejar claro que es una asunción — preséntalo siempre como rango (bajo/medio/alto GB/día) y menciona que es controlable (Basic Logs tier, filtrado de namespaces, sampling).
