# Estimación de Costes — Arquitectura Hub and Spoke con VPN

> **Región de referencia:** West Europe / Spain Central  
> **Moneda:** USD (facturación estándar de Azure; ~0.92 EUR/USD orientativo)  
> **Cálculo:** 730 horas/mes (mes completo)  
> **Supuestos:**
> - 1 Hub VNet + 2 Spoke VNets (mínimo producción)
> - 1 conexión VPN Site-to-Site (enlace on-premises)
> - 1 Azure SQL Database + 1 capacidad Microsoft Fabric
> - Tráfico bajo-medio (~200 GB/mes procesados por Firewall)
> - No incluye licencias de Windows/SQL traídas desde on-premises (AHUB)

---

## 1. Networking — Hub

| Recurso | SKU / Configuración | Precio/hora | Precio mensual |
|---|---|---|---|
| VPN Gateway | VpnGw1 (hasta 650 Mbps, 30 túneles S2S) | $0.19/hr | **$139** |
| VPN Connection S2S | 1 conexión Site-to-Site | $0.015/hr | **$10** |
| Azure Firewall | Standard (inspeción L3-L7, IDPS básico) | $1.25/hr | **$912** |
| Azure Bastion | Basic SKU (RDP/SSH seguro sin IP pública en VMs) | $0.19/hr | **$139** |
| IP Pública — VPN Gateway | Static Standard | $0.005/hr | **$4** |
| IP Pública — Azure Firewall | Static Standard | $0.005/hr | **$4** |
| IP Pública — Azure Bastion | Static Standard | $0.005/hr | **$4** |
| **Subtotal Networking Hub** | | | **$1,212** |

> ℹ️ Azure Firewall es el mayor coste individual de la arquitectura (~48% del total fijo). Es el componente correcto para Hub-Spoke pero algunos entornos de menor criticidad lo sustituyen por NSGs + UDR para reducir costes.

---

## 2. Networking — Spokes

| Recurso | SKU / Configuración | Precio | Precio mensual |
|---|---|---|---|
| Hub VNet | Standard | Gratis | **$0** |
| Spoke VNet 1 | Standard | Gratis | **$0** |
| Spoke VNet 2 | Standard | Gratis | **$0** |
| VNet Peering Hub↔Spoke1 (intra-región) | Bidireccional (2 peerings) | $0.01/GB | ~**$2*** |
| VNet Peering Hub↔Spoke2 (intra-región) | Bidireccional (2 peerings) | $0.01/GB | ~**$2*** |
| Route Tables (UDR) | Para forzar tráfico a Firewall | Gratis | **$0** |
| Network Security Groups | Por subred/NIC | Gratis | **$0** |
| **Subtotal Networking Spokes** | | | **~$4** |

> *Coste variable basado en ~200 GB tráfico inter-spoke. Ver sección de costes variables.

---

## 3. DNS Privado y Seguridad de Identidad

| Recurso | SKU / Configuración | Precio | Precio mensual |
|---|---|---|---|
| Azure Private DNS Zones | 5 zonas (azure.net, database.windows.net, blob.core, vault.azure.net, servicebus) | $0.50/zona/mes | **$3** |
| Azure Key Vault | Standard (secretos, claves, certificados) | ~$0.04/10K ops | **$5** |
| **Subtotal DNS & Seguridad** | | | **$8** |

---

## 4. Observabilidad y Monitorización

| Recurso | SKU / Configuración | Precio | Precio mensual |
|---|---|---|---|
| Log Analytics Workspace | Pay-as-you-go, ~20 GB/mes ingestados | $2.30/GB (tras 5 GB free) | **$35** |
| Azure Monitor — Alertas de métricas | 10 alertas (primeras 10 gratis) | Gratis | **$0** |
| Azure Monitor — Alertas de logs | 5 reglas de alerta de log | $0.10/regla/mes | **$1** |
| Network Watcher | Diagnósticos de red | Gratis (uso básico) | **$0** |
| **Subtotal Observabilidad** | | | **$36** |

---

## 5. Datos — SQL Server + Microsoft Fabric

| Recurso | SKU / Configuración | Precio | Precio mensual |
|---|---|---|---|
| Azure SQL Server (logical server) | Plano de control | Gratis | **$0** |
| Azure SQL Database | General Purpose, 4 vCores provisionados, 32 GB storage incluido | $0.2538/vCore/hr | **$371** |
| SQL Database — Backup Storage | Backup LRS (7 días retención, ~60 GB) | $0.095/GB/mes | **$6** |
| Microsoft Fabric Capacity | **F4 (4 CUs)** — recomendado para SQL Mirroring + Power BI + Pipelines | $0.72/CU/hr | **$526** |
| **Subtotal Datos** | | | **$903** |

> ℹ️ **Microsoft Fabric F2 ($263/mes)** es el mínimo viable; adecuado solo para desarrollo o cargas muy ligeras. Para SQL Mirroring activo + informes Power BI concurrentes + pipelines de datos, **F4 es el punto de partida recomendado para producción**.  
> ℹ️ La capacidad Fabric puede **pausarse fuera de horario laboral** reduciendo hasta un 60% del coste (ej. 12h/día laborables = ~$263/mes en F4).

---

## 6. Almacenamiento Operacional

| Recurso | SKU / Configuración | Precio | Precio mensual |
|---|---|---|---|
| Storage Account (diagnósticos de boot, logs de Firewall) | LRS, Hot tier, ~100 GB | $0.018/GB/mes + transacciones | **$5** |
| **Subtotal Almacenamiento** | | | **$5** |

---

## RESUMEN — Precio Mensual Total Fijo

| Categoría | Precio mensual |
|---|---|
| Networking — Hub | $1,212 |
| Networking — Spokes | $4 |
| DNS Privado & Seguridad | $8 |
| Observabilidad | $36 |
| Datos (SQL + Fabric) | $903 |
| Almacenamiento operacional | $5 |
| **TOTAL MENSUAL ESTIMADO** | **~$2,168 / mes** |
| **TOTAL ANUAL ESTIMADO** | **~$26,016 / año** |

> ⚠️ Este es el coste de **infraestructura base**. No incluye costes variables por consumo ni recursos adicionales.

---

## Costes Variables (Dependen del Uso)

Estos costes fluctúan mes a mes según el consumo real:

| Concepto | Precio unitario | Escenario bajo | Escenario medio | Escenario alto |
|---|---|---|---|---|
| **Egress internet** (datos salientes de Azure) | $0.087/GB (tras 5 GB free) | ~$5 (100 GB) | ~$22 (300 GB) | ~$87 (1 TB) |
| **Firewall — procesado de datos** | $0.016/GB | ~$3 (200 GB) | ~$16 (1 TB) | ~$48 (3 TB) |
| **VNet Peering — transferencia** | $0.01/GB por dirección | ~$2 (200 GB) | ~$10 (1 TB) | ~$30 (3 TB) |
| **SQL — almacenamiento extra** (>32 GB) | $0.115/GB/mes | $0 | ~$7 (60 GB extra) | ~$46 (400 GB) |
| **SQL — Long-Term Backup Retention** | $0.095/GB/mes | ~$5 | ~$19 (200 GB) | ~$57 (600 GB) |
| **Log Analytics — exceso ingesta** | $2.30/GB (>5 GB free) | ~$35 (20 GB) | ~$92 (45 GB) | ~$230 (105 GB) |
| **Key Vault — operaciones** | $0.04/10K ops | ~$1 | ~$3 | ~$10 |
| **Bandwidth VPN** (datos sobre el túnel S2S) | Incluido en tráfico egress | — | — | — |

**Rango variable mensual estimado:** entre **$51** (uso mínimo) y **$508** (uso intensivo).

---

## Recursos Típicos Añadidos a Posteriori

Estos recursos no son necesarios desde el día 1 pero aparecen frecuentemente en la evolución natural de la arquitectura:

### Exposición de Aplicaciones Web

| Recurso | Justificación | SKU típico | Coste estimado/mes |
|---|---|---|---|
| **Azure Application Gateway + WAF** | Balanceo L7 + protección OWASP para apps web internas/externas | WAF_v2, 2 CUs | ~$400 |
| **Azure Front Door Standard** | CDN global + WAF para apps de cara a Internet | Standard | ~$35 + consumo |
| **Azure API Management** | Gobierno de APIs, throttling, autenticación | Developer (no prod) / Standard | $50 / $700 |

### Cómputo y Contenedores

| Recurso | Justificación | SKU típico | Coste estimado/mes |
|---|---|---|---|
| **Azure App Service Plan** | Hosting de aplicaciones web/APIs | P2v3 (2 vCores, 7 GB RAM) | ~$146 |
| **Azure Container Registry** | Almacenamiento privado de imágenes Docker | Standard (100 GB) | ~$20 |
| **Azure Kubernetes Service (AKS)** | Orquestación de contenedores en producción | 3 nodos D4s_v3 | ~$700+ |

### Mensajería y Caché

| Recurso | Justificación | SKU típico | Coste estimado/mes |
|---|---|---|---|
| **Azure Service Bus** | Mensajería asíncrona desacoplada entre servicios | Standard (1 namespace) | ~$10 |
| **Azure Cache for Redis** | Caché de sesiones y datos de alta frecuencia | C1 Standard (1 GB) | ~$55 |
| **Azure Event Hub** | Ingesta de eventos/telemetría a gran escala | Standard, 1 TU | ~$23 |

### Seguridad Avanzada

| Recurso | Justificación | SKU típico | Coste estimado/mes |
|---|---|---|---|
| **Microsoft Defender for Cloud (CSPM)** | Postura de seguridad, recomendaciones | Defender CSPM | $5/servidor/mes |
| **Microsoft Defender for SQL** | Detección de amenazas en base de datos | Standard | $15/instancia/mes |
| **Azure DDoS Protection** | Protección volumétrica de red | Network Protection | ~$2,944 ⚠️ |
| **Entra ID P1** (Azure AD Premium) | MFA, Conditional Access, PIM | Per user | $6/usuario/mes |

> ⚠️ Azure DDoS Network Protection tiene un coste fijo muy elevado ($2,944/mes). Solo se justifica para infraestructuras de alto perfil con riesgo real de ataques volumétricos. Alternativa: DDoS IP Protection (~$199/IP protegida/mes).

### Extensión de Red

| Recurso | Justificación | SKU típico | Coste estimado/mes |
|---|---|---|---|
| **Spoke VNet adicional** | Nuevo entorno (Dev, Staging, DMZ…) | Peering intra-región | $0 + $0.01/GB tráfico |
| **Private Endpoints** (por servicio PaaS) | Eliminar exposición pública de Storage, SQL, KV | $0.01/hr por endpoint | ~$7/endpoint/mes |
| **Azure NAT Gateway** | Salida a Internet controlada desde subnets | Standard | ~$32 |
| **VPN Connection P2S** | Acceso remoto de usuarios (teletrabajo) | Por conexión activa | $0.01/hr por gateway |

### Datos Adicionales

| Recurso | Justificación | SKU típico | Coste estimado/mes |
|---|---|---|---|
| **Azure Data Factory** | Pipelines ETL adicionales fuera de Fabric | Standard, 1K DIU-hrs | ~$10 base + consumo |
| **Azure Synapse Analytics** | Análisis ad-hoc sobre grandes volúmenes | On-demand 100 TB | Variable |
| **Cosmos DB** | Base de datos NoSQL para datos no relacionales | Serverless | ~$25+ uso |

---

## Optimizaciones de Coste Recomendadas

| Optimización | Ahorro estimado | Consideraciones |
|---|---|---|
| **Reserved Instances (1 año)** en VPN Gateway | ~35% = ~$49/mes | Compromiso de 1 año |
| **Reserved Capacity SQL Database (1 año)** | ~33% = ~$122/mes | Compromiso de 1 año |
| **Azure Hybrid Benefit (AHUB)** para SQL | Hasta 40% si tienes licencias SQL Server | Requiere SA activo |
| **Pausa de Fabric Capacity** en no-horario | Hasta 60% en Fabric = ~$316/mes | Solo si no hay cargas nocturnas |
| **Commitment tier Log Analytics** (100 GB/día) | ~15-20% | Solo si superas ~65 GB/día |
| **Azure Firewall Premium → Standard** | Ya en Standard; no degradar | Premium añade IDPS avanzado |

**Ahorro potencial con optimizaciones base (Reservations + AHUB + pausa Fabric):** ~$487/mes

---

## Resumen Ejecutivo

```
Coste fijo base mensual:          ~$2,168 / mes
Costes variables (estimado):       $50 – $500 / mes
─────────────────────────────────────────────────
Rango total mensual estimado:   $2,218 – $2,668 / mes
Rango total anual estimado:    $26,616 – $32,016 / año

Con optimizaciones (reservas + AHUB + pausa Fabric):
  Coste mensual optimizado:      ~$1,681 – $2,181 / mes
```

> **Nota:** Los precios son orientativos en USD (West Europe). La calculadora oficial de Azure ([azure.microsoft.com/pricing/calculator](https://azure.microsoft.com/en-us/pricing/calculator/)) debe usarse para confirmar precios finales antes de la aprobación presupuestaria, ya que pueden variar por región, negociación de contrato Enterprise y cambios de tarifa.

---

*Fuentes de referencia: [VPN Gateway Pricing](https://azure.microsoft.com/en-us/pricing/details/vpn-gateway/) · [Azure Firewall Pricing](https://azure.microsoft.com/en-us/pricing/details/azure-firewall/) · [Azure Bastion Pricing](https://azure.microsoft.com/en-us/pricing/details/azure-bastion/) · [SQL Database Pricing](https://azure.microsoft.com/en-us/pricing/details/azure-sql-database/single/) · [Microsoft Fabric Pricing](https://azure.microsoft.com/en-us/pricing/details/microsoft-fabric/)*
