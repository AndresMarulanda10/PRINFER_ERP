# ERP para Proveedores Industriales — Diseño específico

Resumen ejecutivo
- Objetivo: ERP especializado para proveedores y distribuidores industriales, centrado en cotizaciones B2B, control de márgenes, gestión de pedidos y logística. Diseñado como solución multitenant SaaS y pieza destacada de portafolio.
- Mercados: Colombia y Estados Unidos, con multimoneda (COP/USD), impuestos (IVA/Sales Tax), y envíos nacionales e internacionales.
- Diferenciadores: Portal de clientes B2B, precios personalizados por cliente, control de márgenes, backorder/dropship, integraciones e-commerce B2B y API pública.

1) Concepto (qué resuelve y para quién)
- Personas
  - Ventas (proveedor industrial): necesita generar cotizaciones rápidas con precios y disponibilidad precisos, control de márgenes y seguimiento de pedidos.
  - Compras (proveedor industrial): gestiona relación con sus propios proveedores, POs y recepciones.
  - Almacén/Logística: control de stock, picking, packing, etiquetas y despacho.
  - Administración/Finanzas: AR/AP, impuestos, créditos a clientes, reportes de rentabilidad.
  - Cliente industrial: solicita cotizaciones (RFQ), realiza pedidos, consulta disponibilidad, hace seguimiento y descarga documentos.
- Propuesta de valor
  - Acelera el ciclo quote-to-cash con precios por cliente y control de márgenes.
  - Portal auto-servicio para clientes B2B (reduce fricción comercial).
  - Visibilidad end-to-end: costos, existencias, pedidos, despachos y pagos.
  - Integraciones listas para venta online B2B y envíos.

2) Arquitectura (detallada)
- Estilo: modular por dominios (Core, Customers, Products, Sales, Inventory, Purchases, Finance, Shipping, Integrations).
- Frontend
  - Next.js (App Router) + NextUI/HeroUI + TanStack Query.
  - Portales separados por rol: Interno (staff), Portal Clientes B2B.
  - Internacionalización i18n (es/en), formatos por moneda (COP/USD).
- Backend
  - NestJS: módulos por dominio, DTOs, pipes, guards, interceptors, filtros de excepción.
  - REST v1, versionado por ruta (/api/v1), OpenAPI/Swagger y clients TS generados.
  - Procesos asíncronos: BullMQ + Redis (cotizaciones, PDFs, webhooks, conciliaciones).
- Datos
  - PostgreSQL. Multi-tenant con columna tenant_id en todas las tablas + Row Level Security (RLS).
  - ORM: Prisma o Drizzle. Migraciones versionadas.
  - Almacenamiento de objetos: S3/R2 para catálogos, facturas, guías, imágenes.
- Seguridad
  - Auth: Email/OAuth + organizaciones (Org/Tenant).
  - RBAC: roles internos (admin, ventas, compras, almacén, finanzas) y externos (cliente).
  - OWASP: rate limiting, CSRF en portales, validación de payloads, sanitización, headers de seguridad.
  - Idempotencia en endpoints críticos (creación de órdenes, pagos, webhooks).
- Observabilidad
  - Sentry (errores) + OpenTelemetry (trazas) + logs estructurados (pino).
  - Métricas de negocio: margen por pedido/cliente, fill rate, OTIF a clientes, tiempo de cotización a pedido.
- Integraciones (objetivo)
  - E-commerce B2B: Shopify B2B, WooCommerce B2B, APIs propias de clientes.
  - Envíos: Shippo/EasyPost (global), carriers locales (Servientrega/Coordinadora, UPS/FedEx).
  - Pagos: Stripe Payment Links, pasarelas locales (Colombia).
  - Fiscalidad: interfaces para DIAN (Colombia) y sales tax (US).
- Despliegue/Flexibilidad
  - Web: Vercel (SSR/ISR y previews).
  - API/Jobs: Fly.io/Railway/Render.
  - DB: Neon/Supabase/Postgres gestionado. Backups diarios, PITR.
  - Infra como código (fase posterior): Terraform.
- No funcionales
  - Performance: p95 < 300 ms en consultas típicas (listas paginadas con filtros indexados).
  - Concurrencia: 50–200 req/s objetivos iniciales; crecimiento horizontal con réplicas.
  - SLO: uptime 99.5% MVP; error budget y alertas básicas.

3) Módulos y modelo de datos (MVP y extensiones)
- Core
  - Organization (tenant_id, nombre, plan, dominio), User, Membership (rol), AuditLog (actor, acción, entidad, diff, timestamp).
  - Config: moneda base por tenant, impuestos, UoM, zonas horarias.
- Customers (Clientes B2B)
  - Customer(id, tenant_id, name, contact_info, credit_limit, payment_terms, tax_id).
  - CustomerPriceList(id, tenant_id, customer_id, active).
  - CustomerUser(id, customer_id, email, role) para acceso al portal.
- Products (Catálogo)
  - Product(id, tenant_id, sku único por tenant, name, description, attributes JSON, cost).
  - ProductPrice(id, tenant_id, product_id, price_list_id, min_qty, price) para precios por volumen/cliente.
  - Kit(id, tenant_id, sku, components[]) opcional para productos ensamblados.
  - Media/Attachment(opcional: imágenes, fichas técnicas).
- Sales (Ventas)
  - Quote(id, tenant_id, customer_id, status: Draft|Sent|Accepted|Rejected, valid_until, lines[], totals).
  - QuoteLine(id, quote_id, product_id, qty, unit_price, discount, tax, margin).
  - SalesOrder(id, tenant_id, customer_id, quote_id?, status: Confirmed|Allocated|Partial|Shipped|Invoiced|Completed).
  - SalesOrderLine(id, so_id, product_id, qty, allocated_qty, shipped_qty, unit_price, tax).
  - CustomerInvoice(id, tenant_id, customer_id, so_id, amount, tax, status: Draft|Sent|Paid).
- Inventory (Inventario)
  - Warehouse(id, tenant_id, name, address, country).
  - Stock(id, tenant_id, product_id, warehouse_id, qty_on_hand, qty_allocated, qty_available).
  - InventoryTx(id, tenant_id, type: IN|OUT|ADJ|TRANSFER, qty, refs: GRN/SO/SHIP, created_by).
  - Allocation(id, tenant_id, so_line_id, product_id, warehouse_id, qty).
- Shipping (Envíos)
  - Shipment(id, tenant_id, so_id, carrier, tracking_number, status, labels[]).
  - ShipmentLine(id, shipment_id, so_line_id, product_id, qty).
  - RateQuote(carrier, service, cost, eta).
- Purchases (Compras a proveedores)
  - Supplier(id, tenant_id, name, contact_info).
  - PurchaseOrder(id, tenant_id, supplier_id, status, linked_so_id?, is_dropship).
  - PurchaseOrderLine(id, po_id, product_id, qty, price, linked_so_line_id?).
  - GoodsReceipt(id, po_id, lines[]) para recepción de mercancía.
  - SupplierInvoice(id, supplier_id, po_id, amount).
- Finance (Finanzas)
  - ARTx(id, tenant_id, customer_id, amount, type: Invoice|Payment|Credit|Debit).
  - APTx(id, tenant_id, supplier_id, amount, type: Invoice|Payment).
  - Tax(id, tenant_id, code, rate, country, region).
  - CurrencyRate(from_currency, to_currency, rate, date).
- Integrations
  - ConnectorConfig(Shopify/WooCommerce/Shippo), APIKey, WebhookSubscription, Outbox/Event.

4) Funcionalidades (específicas con criterios)
- Portal de Clientes B2B
  - Registro por invitación: cliente recibe email, completa perfil, accede a catálogo con precios personalizados.
  - Solicitar cotización (RFQ): catálogo con precios específicos + cantidades → genera Quote en borrador para revisión interna.
  - Pedidos: ver histórico, estado en tiempo real, descargar documentos (cotización, SO, factura, guías).
  - Pagos: enlaces de pago, estado de cuenta, crédito disponible.
- Ventas y cotizaciones
  - Pricing avanzado: precios base, listas por cliente, descuentos por volumen.
  - Cotizador con control de margen: simulación, alertas por debajo de mínimo, aprobaciones.
  - Flujo Quote → SO → Allocation → Shipment → Invoice con trazabilidad completa.
  - Backorder y dropship automáticos según configuración y disponibilidad.
- Inventario
  - Multi-almacén con transferencias, reservas por SO.
  - Reglas de asignación (FIFO/FEFO, almacén preferido por cliente/región).
  - Min/Max, punto de reorden, alerta de stock bajo.
- Envíos
  - Selección de carrier (integraciones), impresión de etiquetas/guías.
  - Notificaciones automáticas a clientes, tracking en portal.
- Compras a proveedores
  - Generación automática desde SO (bajo mínimo o dropship).
  - Recepción, actualización de costos y disponibilidad.
- Finanzas
  - AR/AP con términos de pago (Net 30/60).
  - Límites de crédito, retenciones de impuestos.
- Reportes
  - Rentabilidad por cliente/producto/pedido.
  - Fill rate, OTIF, rotación de inventario.
  - Forecast basado en histórico.

Criterios de aceptación (ejemplos)
- Cotización a cliente
  - Dado un vendedor con catálogo y precios por cliente, cuando genera una cotización, entonces puede simular márgenes, enviar PDF al cliente y recibir aceptación en portal.
- Control de margen
  - Dado un precio de costo y regla de margen mínimo por categoría, cuando el vendedor intenta cotizar por debajo del mínimo, entonces el sistema alerta y requiere aprobación de nivel superior.
- Pedido y asignación
  - Dado un SO confirmado, cuando el sistema asigna stock, entonces reserva inventario específico, genera picking list y actualiza el estado "allocated".

5) API (endpoints clave del MVP)
- Customers & Pricing
  - GET /api/v1/customers — lista con filtros (name, status).
  - GET /api/v1/customers/:id/prices — precios específicos del cliente.
  - POST /api/v1/customers/:id/users — invitar usuarios al portal.
- Sales
  - POST /api/v1/quotes — crear cotización.
  - POST /api/v1/quotes/:id/send — enviar al cliente (email + portal).
  - POST /api/v1/quotes/:id/convert — convertir a SO.
  - GET /api/v1/orders?status=&customer=&date= — listar con filtros.
- Inventory
  - GET /api/v1/inventory/stock?product=&warehouse= — disponibilidad.
  - POST /api/v1/inventory/allocate — asignar a SO.
  - GET /api/v1/inventory/transactions — movimientos con trazabilidad.
- Shipping
  - POST /api/v1/shipments/rates — cotizar envío (carriers).
  - POST /api/v1/shipments — crear envío y generar etiquetas.
- Portal de Clientes (API pública)
  - GET /api/v1/portal/catalog — catálogo con precios del cliente.
  - POST /api/v1/portal/quote-requests — solicitar cotización.
  - GET /api/v1/portal/orders — pedidos del cliente.

Notas de diseño API
- Autenticación: Bearer JWT o sesión; scoping por tenant.
- Paginación: cursor o page/size; ordenar; filtros consistentes.
- Idempotencia: header Idempotency-Key en POST sensibles.
- Errores: formato uniforme { code, message, details }.

6) RBAC (matriz de permisos específica)
- Internos
  - admin: acceso completo del tenant (incluye facturación SaaS).
  - ventas: CRUD Quotes/SO/Customers, lectura Inventory, crear Shipments.
  - almacén: lectura SO, crear/modificar Allocations/Shipments, ajustes de Stock.
  - compras: CRUD Suppliers/PO/GRN, lectura Inventory/Products.
  - finanzas: AR/AP/Invoices, lectura SO/PO, reportes, límites de crédito.
- Externos
  - cliente (portal): ver su catálogo con precios, crear RFQs, ver SOs propios, tracking, descargar documentos propios.
  - cliente_admin: lo anterior + invitar usuarios, límites de compra internos.

7) Gestión ágil (SCRUM-Ban con detalle)
- Ceremonias
  - Planning (cada 2 semanas, 90 min): objetivo del sprint, selección de historias, definición de capacidad.
  - Daily (15 min): bloqueos, foco de día, revisión de WIP.
  - Review (30–45 min): demo funcional de incrementos.
  - Retro (30 min): acciones concretas de mejora (máx 1–2 por sprint).
  - Grooming (semanal, 45–60 min): refinar historias, criterios, estimación (story points).
- Tablero Kanban (WIP estricto)
  - Columnas: Backlog → Ready → In Progress (WIP 2/dev) → Code Review (WIP 2) → Testing/QA → Done.
  - Políticas: nada entra a In Progress sin criterios de aceptación y diseño de datos/API.
- Definiciones
  - Definition of Ready (DoR): historia con user story clara, criterios, diseño de API/datos bosquejado, dependencias identificadas.
  - Definition of Done (DoD): criterios pasados, tests básicos, manejo de errores, logs, docs API actualizadas, feature usable y demo en staging.
- Estimación y métricas
  - Estimar con Fibonacci (1,2,3,5,8). Medir throughput, lead-time, WIP aging.
  - Objetivo: 1–2 incrementos demoables por sprint que aporten valor visible.

8) Roadmap por sprints (específico)
- Sprint 1: Fundaciones + Catálogo con precios por cliente + Cotizaciones
  - Setup repositorio, base multitenant, RBAC inicial.
  - Importación de productos, configuración de precios por cliente.
  - Cotizador con control de margen y envío al cliente (PDF/email).
  - Entregables: demo "selecciono cliente → catálogo muestra precios específicos → genero cotización con alertas de margen → envío PDF → trazabilidad".
- Sprint 2: Pedidos + Inventario + Asignación
  - Flujo Quote → SO, asignación de stock, reglas de backorder/dropship.
  - Estado de pedido en tiempo real, asignación manual/automática.
  - Entregables: demo "Quote aceptada → SO → asignación de stock → picking list → estado actualizado y visible al cliente".
- Sprint 3: Portal de Clientes B2B + Envíos
  - Portal para clientes: catálogo personalizado, RFQs, seguimiento de pedidos.
  - Integración básica de envíos (rate shopping, etiquetas).
  - Entregables: demo "cliente solicita cotización en portal → vendedor procesa → cliente acepta → tracking".

Backlog detallado (historias ejemplo con criterios)
- Épica: Pricing y Cotizaciones
  - Historia: como vendedor, configuro precios específicos por cliente.
    - Criterios: crear lista base, asignar a cliente, override de SKUs específicos, validar coherencia.
  - Historia: como vendedor, genero cotización con control de margen.
    - Criterios: alerta si margen < mínimo por categoría, requiere aprobación para continuar, simula impacto de descuentos.
- Épica: Portal de Clientes
  - Historia: como cliente, solicito cotización (RFQ) desde el portal.
    - Criterios: veo mi catálogo con mis precios, añado cantidades, genero solicitud, recibo confirmación, veo estado.
  - Historia: como cliente, acepto cotización y convierto a pedido.
    - Criterios: veo validez, T&C, confirmo, genera SO en sistema.
- Épica: Gestión de pedidos
  - Historia: como vendedor, gestiono pedidos con asignación de stock.
    - Criterios: veo disponibilidad, asigno manual o automático, genero picking list, cambio estados.
  - Historia: como operador de almacén, preparo pedidos con guía visual.
    - Criterios: lista de picking por ubicación, confirmación de cantidades, validación.
- Épica: Envíos
  - Historia: como operador logístico, genero etiquetas y guías.
    - Criterios: selecciono carrier, dimensiones, genero PDF/ZPL, registro tracking.
- Épica: Reportes
  - Historia: como gerente, veo rentabilidad por cliente/producto.
    - Criterios: filtros por período, drill-down, export Excel/PDF.

9) Riesgos y mitigaciones
- Complejidad de pricing B2B: modelo flexible pero con defaults sensatos; validación de coherencia para evitar errores de precio.
- Integración con carriers variables por país: abstracción por carrier, mapeo de servicios, cache de tarifas.
- Límites de crédito y términos de pago: workflows de aprobación, alertas, bloqueo configurable.
- Escalabilidad multi-tenant: RLS en Postgres, particionado futuro por tenant_id, índices específicos.

10) Estrategia de pruebas y datos
- Unit tests: servicios de dominio (cálculo de precios, márgenes, asignaciones), validaciones.
- Integration tests: endpoints críticos (cotización, pedido, asignación).
- E2E liviano: flujos clave con datos semilla (Playwright).
- Datos semilla realistas: 2 tenants demo, 10 clientes, 500 SKUs con precios variables, 50 quotes/SOs en distintos estados.

11) "Wow features" planificadas
- Simulador de márgenes con what-if para descuentos y promociones.
- Sugeridor de cross-sell/up-sell basado en histórico de pedidos.
- Calculador de lead time dinámico basado en histórico de proveedores.
- Integración con plataformas de procurement de clientes (Ariba, Coupa) vía conectores.
- App móvil (Expo) para vendedores en campo y operadores de almacén.
- Dashboard personalizado por rol con KPIs críticos.

12) Métricas y KPIs (definiciones)
- Tiempo de cotización: solicitud RFQ → envío de Quote.
- Tasa de conversión: Quotes enviadas → SOs confirmados.
- Fill rate: líneas servidas completas / líneas pedidas.
- OTIF a clientes: On-Time In-Full / total pedidos.
- Rotación de inventario: Costo de ventas / inventario promedio.
- Margen promedio por cliente/categoría.

Apéndice: índices recomendados
- products: (tenant_id, sku) unique; (tenant_id, name) gin_trgm opcional.
- product_price: (tenant_id, product_id, price_list_id, min_qty).
- stock: (tenant_id, product_id, warehouse_id).
- sales_order: (tenant_id, customer_id, status, created_at).
- allocation: (tenant_id, so_line_id); (tenant_id, product_id, warehouse_id).
- inventory_tx: (tenant_id, created_at, type).
