# Mosca ERP v2 — Arquitectura y Plan de Implementación
**Versión:** 1.3 · 2026-05-30  
**Estado:** Aprobado — listo para implementar

---

## 1. Contexto y principios

### El problema actual
El sistema actual creció modelando el negocio ideal en lugar del negocio real. El resultado es una base de datos con ~300 campos activos donde el 60% genera fricción sin agregar valor: tablas duplicadas que hay que sincronizar manualmente, lookups masivos como sustituto de una interfaz propia, lógica de costeo completa sin datos para alimentarla.

Las consecuencias concretas:
- La carga de movimientos se posterga porque es lenta y confusa
- El dashboard muestra datos incorrectos (campo `tipo_flujo` ausente en Gastos, registros faltantes)
- Producción y Pedidos son dos tablas que modelan el mismo objeto → doble mantenimiento
- Nadie del equipo actualiza estados porque no hay una interfaz pensada para ellos

### Principios de diseño para v2

**Modelar el negocio de hoy, no el de dentro de 2 años.** Recetas, stock de materiales y costeo por variante son poderosos pero prematuros. Se archivan.

**Interfaces dedicadas, no vistas de Airtable.** Los lookups masivos existen porque Airtable es el frontend. Cuando cada rol tiene su pantalla, la base de datos se simplifica.

**Un objeto, un registro.** Un pedido es un pedido desde que entra hasta que se cobra. No hay Pedido + Produccion — hay un Pedido con estados.

**Cada servicio tiene una responsabilidad.** Los workflows de n8n son microservicios: una entrada, una salida definida, sin lógica mezclada. Esto los hace reutilizables y documentables para la agencia.

**Crear de cero, migrar datos.** No se parchea la base existente. Se construye la nueva base limpia y se migran solo los datos activos necesarios.

---

## 2. Arquitectura general

```
[Kommo CRM]          [WhatsApp / texto libre]     [Candela · celular]
     │                        │                          │
     ▼                        ▼                          ▼
[n8n: kommo-sync]    [n8n: ingesta-movimientos]   [Taller Interface]
     │                        │                          │
     └──────────────┬──────────┘                         │
                    ▼                                     ▼
             [Airtable v2]  ◄──────────────────────────────
             (fuente de verdad)
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
   [Dashboard    [Dashboard  [n8n: alertas
    Finanzas]    Operaciones] diarias]
```

**Stack sin cambios:**
- Airtable Pro → base de datos
- n8n → automatizaciones y orquestación
- GitHub Pages → hosting de frontends HTML
- OpenRouter (Claude) → parseo de lenguaje natural
- Kommo CRM → pipeline comercial (sin tocar)

---

## 3. Modelo de datos v2

### 3.1 Tablas que se conservan (simplificadas)

#### Clientes
Sin cambios. Está bien como está.

| Campo | Tipo | Notas |
|-------|------|-------|
| telefono | singleLineText | PK natural |
| nombre | singleLineText | |
| email | email | |
| direccion | singleLineText | |

*(Se elimina el link a Produccion — ya no existe esa tabla)*

---

#### Pedidos
Se fusiona con Produccion. Un pedido tiene estados comerciales y estados de taller separados.

| Campo | Tipo | Notas |
|-------|------|-------|
| id_pedido | autoNumber | **Este es el ID de la ficha física del taller** |
| tipo | singleSelect | **Cliente / Stock** — Stock = producción propia o devolución disponible para venta |
| cliente | link → Clientes | vacío si tipo = Stock |
| fecha_seña | date | |
| fecha_entrega_comprometida | formula | `fecha_seña + 30 + dias_adicionales` |
| dias_adicionales | number | puede ser negativo para acortar |
| precio_pactado | currency | precio final negociado, incluye todo (envío, instalación si aplica) |
| estado_comercial | singleSelect | Señado / En producción / Para entregar / Entregado / Cobrado |
| notas_logistica | multilineText | dirección, tipo de acceso, piso, si requiere envío, observaciones de entrega |
| tono_madera | singleSelect | |
| terminacion | singleSelect | |
| tono_hierro | singleSelect | |
| tipo_madera | singleSelect | |
| acabado | singleSelect | |
| catalogo | link → Catalogo | opcional — referencia al producto del catálogo si aplica |
| ingresos | link → Ingresos | para calcular saldo |
| Resultados | link → Resultados | mes del pedido |

**Sobre el ID de ficha:** El `id_pedido` es el identificador que se usa en el taller para las fichas físicas. Todos los muebles de un mismo pedido comparten ese número. Los items individuales se identifican dentro de la ficha como líneas (`id_pedido` + número de línea, ej: "42-1", "42-2").

**Sobre precio_pactado:** Es el monto total acordado con el cliente. No hay campos separados de envío e instalación — si el pedido incluye esos servicios, el monto ya está incorporado. Las condiciones de entrega van en `notas_logistica`.

*Campos eliminados vs v1: `envio`, `instalacion`, `premio_edgar`, todos los lookups de producción, costo_materiales_pedido, precio_pedido_promedio, monto_total_promocionado, y otras fórmulas que dependen de Variantes/Recetas.*

---

#### Items_Pedido *(tabla nueva)*
Reemplaza la tabla Produccion. Un registro por mueble dentro de un pedido. Esta es la tabla que usa Candela.

| Campo | Tipo | Notas |
|-------|------|-------|
| id_item | autoNumber | sub-identificador interno |
| nro_linea | number | 1, 2, 3… dentro del pedido — para mostrar "42-1", "42-2" en la ficha |
| pedido | link → Pedidos | |
| catalogo | link → Catalogo | referencia al ítem del catálogo |
| nombre | lookup (catalogo.nombre) | nombre del producto — no se escribe, viene del catálogo |
| dimensiones | singleLineText | texto libre, ej "120×60×45 cm" |
| estado | singleSelect | **Nuevo / En herrería / En carpintería / En pintura / Para entregar / Entregado / En Stock / Vendido / Rehacer / Cancelado** |
| fecha_entregado | date | se completa al entregar |
| notas_taller | multilineText | problemas, observaciones |
| imagen | attachment | foto del mueble terminado (opcional) |

**Estados del taller** (tomados del proceso real):
- `Nuevo` — llegó el pedido, aún no arrancó
- `En herrería` — en proceso en herrería
- `En carpintería` — en proceso en carpintería  
- `En pintura` — en proceso de pintura/terminación
- `Para entregar` — terminado, listo para salir
- `Entregado` — en manos del cliente
- `En Stock` — disponible para venta inmediata (producción propia o devolución/cancelación)
- `Vendido` — era stock y fue vendido a un cliente (se creó un nuevo Pedido de Cliente)
- `Rehacer` — hay un problema, se rehace
- `Cancelado` — item cancelado

**Lógica de stock:** Los pedidos con `tipo = Stock` son producciones internas sin cliente. Sus items en estado `En Stock` son visibles en el cotizador de Alumine con precio y dimensiones, marcados como "entrega inmediata". Cuando Alumine cierra una venta de un item en stock: se crea un Pedido de Cliente normal, el item cambia a `Vendido`.

*Sin lookups. Sin fórmulas. Sin links a Variantes. La interfaz de Candela lee solo esta tabla.*

---

#### Ingresos
Se agrega el campo `tipo_flujo` que hoy falta.

| Campo | Tipo | Notas |
|-------|------|-------|
| id_ingreso | autoNumber | |
| fecha | date | String en n8n (ver lecciones técnicas) |
| tipo_flujo | singleSelect | **Operativo / Capital** ← campo nuevo explícito |
| concepto | singleSelect | Seña, Saldo, Venta, Recupero Edgar, Capitalizacion - Lisandro, Capitalizacion - Javier, Recupero capital Lisandro, Recupero capital Javier, Venta Eldra |
| Nota | singleLineText | descripción libre |
| monto | currency | |
| metodo_cobro | singleSelect | |
| pedido | link → Pedidos | |
| comprobante | attachment | |
| Resultados | link → Resultados | |

---

#### Gastos
Se agrega `tipo_flujo` explícito. Se elimina `Resultados Mensuales` (campo texto redundante).

| Campo | Tipo | Notas |
|-------|------|-------|
| id_gasto | autoNumber | |
| fecha | date | |
| tipo_flujo | singleSelect | **Operativo / Capital** ← campo nuevo explícito |
| concepto | singleLineText | |
| categoria | singleSelect | Materiales, Mano de Obra, Alquiler, Servicios, Marketing, Herramientas y Equipos, Reintegro, Otros, Prestamos Edgar |
| tipo | singleSelect | Fijo / Variable / Única vez |
| monto | currency | |
| metodo_pago | singleSelect | |
| Proveedores | link → Proveedores | |
| comprobante | url | |
| Resultados | link → Resultados | |

---

#### Resultados *(simplificado)*
Se mantiene pero reducida a tabla mínima. Su único rol es ser un "mes" al que linkan Ingresos, Gastos y Pedidos, facilitando queries por período.

| Campo | Tipo | Notas |
|-------|------|-------|
| Mes | singleLineText | formato YYYY-MM, ej "2026-05" |

**Se eliminan todas las fórmulas y lookups.** Los cálculos de totales mensuales los hace el dashboard leyendo directamente Ingresos y Gastos filtrados por el link a Resultados. Airtable deja de ser el lugar donde viven los agregados — eso pasa al frontend.

---

#### Proveedores
Sin cambios. Simple y funcional.

---

#### Catalogo *(renombrada desde Variantes)*
El cotizador de Alumine necesita un catálogo de productos con precios de referencia. Esta tabla lo provee. Se renombra y simplifica radicalmente — deja de ser un sistema de costeo y pasa a ser un catálogo de precios manual.

| Campo | Tipo | Notas |
|-------|------|-------|
| nombre | singleLineText | "Mesa ratona", "Silla", "Biblioteca" |
| descripcion | multilineText | descripción del producto |
| linea | singleSelect | línea de producto |
| dimensiones_referencia | singleLineText | texto libre, ej "120×60×45 aprox" |
| precio_lista | currency | precio de referencia — se actualiza manualmente |
| precio_promocional | currency | precio promo si aplica |
| imagen | attachment | foto de referencia |

**Sobre productos custom:** Si el pedido es un producto que no existe en el catálogo, el campo `catalogo` del Pedido queda vacío y el `precio_pactado` se ingresa libremente. No es necesario crear una entrada de catálogo para cada encargo custom.

**Campos eliminados de Variantes:** horas_mano_obra, costo_materiales, costo_total, costo_mano_de_obra, costos_fijos, margen_ganancia, precio_superpromo, rel_materiales_precio, largo/alto/ancho/profundidad_cm (pasan a dimensiones_referencia como texto), todos los links a Recetas, todos los lookups.

---

### 3.2 Tablas que se archivan

Estas tablas se exportan a CSV y se eliminan de la base activa. Se pueden restaurar cuando sea relevante.

- **Produccion** → reemplazada por Items_Pedido
- **Variantes** (v1) → reemplazada por Catalogo simplificado
- **Recetas** → prematura para el estadio actual
- **Materiales** → sin stock activo, sin cotizador por costo real
- **Productos** → los productos relevantes migran a Catalogo
- **Eldra** → tiene su propia base, no pertenece acá

### 3.3 Resumen de tablas v2

| Tabla | Rol | Quién la usa |
|-------|-----|--------------|
| Clientes | Directorio de clientes | Sistema |
| Pedidos | Pedido desde seña hasta cobro | Kommo-sync, dashboard, Lisandro |
| Items_Pedido | Muebles individuales del pedido | Candela (taller), dashboard |
| Catalogo | Precios de referencia | Cotizador Alumine |
| Ingresos | Movimientos de entrada de dinero | Ingesta-movimientos |
| Gastos | Movimientos de salida de dinero | Ingesta-movimientos |
| Proveedores | Directorio de proveedores | Gastos |
| Resultados | Índice de meses (foreign key) | Ingresos, Gastos, Pedidos |

---

## 4. Servicios (microservicios n8n)

Cada servicio es un workflow n8n independiente. Contrato definido: entrada conocida → salida conocida. Sin lógica mezclada entre servicios.

---

### Servicio 1: `ingesta-movimientos`
**Descripción:** Parseo de texto libre → registro en Airtable  
**Estado:** En producción (v0.4). Estabilizar y documentar.

**Entrada:** POST con `{ texto, fecha? }`  
**Proceso:** Webhook → Code node ($http a OpenRouter/Claude) → If tipo → nodo Airtable Ingresos o Gastos  
**Salida:** `{ ok: true, record_id, concepto, monto, tipo_flujo, posible_duplicado }`  
**Reglas clave:**
- CBU 4530000800019697462992 (Naranja X Lisandro) → siempre Ingreso Operativo
- `fecha` como String en nodo Airtable (nunca dateTime)
- `Resultados` como array: `[$json.resultado_id]` o `undefined`
- Mapa Resultados → hardcodeado por ahora, migrar a lookup dinámico en v2

**Cambios para v2:** Adaptar para que lea `tipo_flujo` del nuevo campo explícito. Actualizar mapa de Resultados con los IDs de la nueva base.

---

### Servicio 2: `kommo-sync`
**Descripción:** Evento de venta en Kommo → Pedido + Cliente en Airtable  
**Estado:** Parcial. Crea Cliente y registra Seña. No crea Items_Pedido.

**Entrada:** Webhook Kommo evento "Señado" (lead ganado / etapa específica del pipeline)  
**Proceso:** Webhook → extraer campos del lead → upsert Cliente → crear Pedido → crear Items_Pedido (uno por mueble si viene la info, sino uno genérico)  
**Salida:** `{ pedido_id, cliente_id, items_creados }`  

**Campos que llegan de Kommo:** nombre cliente, teléfono, monto pactado, notas de la conversación, fecha  
**Campos que NO llegan (se completan luego):** specs físicas (tono, madera, medidas), items específicos  

**Lógica de items:** Si Kommo trae descripción del mueble en el campo de notas → crear item con esa descripción. Si no → crear un item placeholder "Mueble sin especificar" para que el taller lo complete.

---

### Servicio 3: `taller-interface`
**Descripción:** Interfaz mobile-first para que Candela actualice estados de producción  
**Estado:** A construir.

No es un workflow de n8n — es un frontend HTML en GitHub Pages que lee y escribe Airtable directamente vía API REST.

**Pantalla principal:** Lista de Items_Pedido agrupados por Pedido, ordenados por fecha_entrega_comprometida ASC. Muestra: descripción del mueble, cliente, fecha comprometida, estado actual.

**Acción:** Tap en un item → modal con los 4 estados → seleccionar → guarda directo en Airtable.

**Acceso:** URL con token en query param (`?token=xxx`). Sin login complejo. Token rotable.

**Restricciones:** Solo ve Items_Pedido y datos mínimos del Pedido asociado. No ve datos financieros. No puede crear ni eliminar pedidos.

---

### Servicio 4: `dashboard-operaciones`
**Descripción:** Vista del estado del negocio para el equipo  
**Estado:** A construir. Reemplaza la parte operativa del dashboard actual.

**Acceso:** Público dentro del equipo (sin PIN). Disponible en celular y tablet.

**Contenido:**
- Pedidos activos: tabla con cliente, descripción, fecha entrega comprometida, estado_comercial, saldo pendiente
- Items atrasados: items con fecha_entrega_comprometida < hoy y estado != Entregado, destacados en rojo
- Próximas entregas: pedidos con entrega en los próximos 7 días
- Kanban simple: columnas por estado de items (Pendiente / En proceso / Listo / Entregado)

---

### Servicio 5: `dashboard-finanzas`
**Descripción:** Números del negocio para Lisandro y Javier  
**Estado:** A construir. Reemplaza el dashboard financiero actual (con datos corregidos).

**Acceso:** PIN de 4 dígitos. Solo Lisandro y Javier.

**Contenido:**
- **Resultado del mes:** ingresos operativos, gastos operativos, resultado neto
- **Posición de caja estimada:** suma de todos los ingresos menos todos los gastos (operativos + capital)
- **Por cobrar:** pedidos entregados con saldo > 0
- **Cuentas corrientes socios:** posición Lisandro y Javier (capitalizaciones vs recuperos)
- **Cuenta corriente Edgar:** DEBE (Prestamos Edgar) vs HABER (Recupero Edgar + devoluciones + premios). Saldo histórico completo.
- **Evolución mensual:** últimos 6 meses, ingresos vs gastos
- **Movimientos filtrables:** tabla de ingresos y gastos con filtros por mes, tipo, categoría

**Reglas contables:**
- Ingresos Operativos = `tipo_flujo = Operativo`
- Capitalizaciones = `tipo_flujo = Capital`, concepto = Capitalización
- Resultado neto = Ingresos Operativos − Gastos Operativos (sin Capital)
- CC Edgar: DEBE = Gastos categoria=Prestamos Edgar; HABER = Ingresos concepto=Recupero Edgar + concepto=Préstamos con nota "edgar" + premios_edgar de Pedidos

---

### Servicio 6: `alertas`
**Descripción:** Chequeo diario automatizado → notificación WhatsApp  
**Estado:** A construir.

**Schedule:** 9am todos los días hábiles  
**Proceso:** n8n lee Airtable → construye resumen → envía WhatsApp a Lisandro

**Alertas:**
- Items con estado != Entregado y fecha_entrega_comprometida <= hoy → "X items atrasados"
- Items sin actualizar hace más de 3 días (fecha de última modificación) → "X items sin movimiento"
- Pedidos con saldo pendiente y estado = Entregado → "X pedidos sin cobrar"
- Pedidos señados hace más de 30 días sin Items en estado "En proceso" → "X pedidos estancados"

**Canal:** WhatsApp Business API o Twilio (a definir según disponibilidad).

---

## 5. Plan de migración e implementación

### Principio de migración
Las tablas v2 se crean **dentro de la misma base existente** (`appBThGsxqCSljqH1`) con nombres nuevos. Conviven con las tablas v1 durante la transición. Cuando todo funciona, las tablas v1 se archivan (se ocultan o eliminan). No hay migración de base, solo de datos entre tablas dentro de la misma base.

**Ventaja:** los links entre tablas (ej: Ingresos → Resultados) siguen funcionando con los mismos IDs de registro. No hay que migrar todo — solo los datos activos a las tablas nuevas.

**Dato activo = registros de los últimos 12 meses + pedidos con estado != Cobrado.**

**Tablas v1 NO se renombran.** Los workflows de n8n y el dashboard dependen de esos nombres — cambiarlos rompe producción. Las tablas nuevas conviven con sufijo `_v2`: `Pedidos_v2`, `Items_Pedido`, `Catalogo`, `Resultados_v2`. Cuando todo esté migrado y funcionando, se decide qué hacer con las v1.

---

### Fase 0 — Setup del schema (1-2 días)
*Crear las tablas nuevas en la base existente. No se toca ningún dato.*

1. Renombrar `Pedidos` → `Pedidos_v1` y `Produccion` → `Produccion_v1` en Airtable (manual, 2 min)
2. Crear tablas nuevas vía MCP: `Resultados_v2`, `Pedidos`, `Items_Pedido`, `Catalogo`
3. Agregar campo `tipo_flujo` a `Ingresos` y `Gastos` existentes
4. Crear registros de `Resultados_v2` para los meses 2025-06 a 2026-12
5. Documentar todos los IDs nuevos (tablas + registros de Resultados)

**Criterio de avance:** Las tablas nuevas existen con el schema correcto. Los IDs de Resultados_v2 están documentados.

---

### Fase 1 — Taller Interface (1 semana)
*Primera prioridad porque es la que más impacta la calidad de datos.*

1. Construir `taller-interface` HTML (GitHub Pages)
2. Conectar con Airtable v2 vía API (token en URL)
3. Probar con Candela en celular
4. Ajustar UX según feedback
5. Deploy y comunicar al equipo

**Criterio de avance:** Candela puede cambiar estados desde el celular en menos de 30 segundos por item.

---

### Fase 2 — Servicios financieros (1-2 semanas)
*Reemplaza el dashboard actual con datos correctos.*

1. Migrar `ingesta-movimientos` a nueva base (actualizar IDs de Resultados, adaptar tipo_flujo)
2. Construir `dashboard-finanzas` con las reglas contables de §4
3. Construir `dashboard-operaciones`
4. Validar cálculos contra registros conocidos (comparar con registros manuales)
5. Deploy

**Criterio de avance:** Los números del dashboard coinciden con los que Lisandro puede calcular a mano para un mes cerrado conocido.

---

### Fase 3 — Kommo sync completo (1 semana)
*Cierra el loop de entrada de datos desde ventas.*

1. Actualizar workflow `kommo-sync` para apuntar a nueva base
2. Agregar creación de Items_Pedido desde datos de Kommo
3. Probar con una venta real en staging
4. Deploy y monitorear primera semana

**Criterio de avance:** Una venta que entra por Kommo genera automáticamente Cliente + Pedido + Item(s) en Airtable sin intervención manual.

---

### Fase 4 — Alertas (3-5 días)
*Una vez que los datos son confiables, las alertas tienen sentido.*

1. Construir workflow `alertas` en n8n
2. Definir canal de notificación (WhatsApp / Telegram / email)
3. Configurar schedule
4. Ajustar umbrales de alerta según experiencia real

---

### Fase 5 — Documentación para agencia (ongoing)
*Corre en paralelo desde Fase 2 en adelante.*

Para cada servicio, documentar:
- **Nombre del servicio** y responsabilidad única
- **Contrato de entrada/salida** (campos exactos, tipos, validaciones)
- **Diagrama de flujo** del workflow n8n
- **Dependencias** (qué tablas lee/escribe, qué APIs consume)
- **Costo de implementación estimado** (horas, herramientas)
- **Variables de configuración** para reutilización en otro cliente (IDs, tokens, umbrales)

El objetivo es que cada servicio pueda instalarse en otro negocio cambiando solo las variables de configuración.

---

## 6. Decisiones pendientes

Estos puntos requieren decisión antes de implementar las fases correspondientes:

| # | Decisión | Opciones | Impacta |
|---|----------|----------|---------|
| 1 | Canal de alertas | WhatsApp Business API / Twilio / Telegram / Email | Fase 4 |
| 2 | Mapa de Resultados v2 | Hardcodeado en HTML vs dinámico desde Airtable | Fase 2 (ingesta) |
| 3 | Cotizador de Alumine | Sigue leyendo v1 durante transición / migra en Fase 2 | Fase 0 |
| 4 | Acceso dashboard finanzas | PIN en frontend vs token en URL | Fase 2 |
| 5 | Items_Pedido desde Kommo | Descripción libre desde notas / placeholder único | Fase 3 |

---

## 7. Lo que NO se hace en v2

Para que quede claro el alcance:

- **Stock de materiales** — no se implementa. Se puede agregar en v3 cuando sea relevante.
- **Costeo por variante (Recetas)** — archivado. Útil para cotizador avanzado en v3.
- **Contabilidad formal** — este sistema no reemplaza un software contable. Es gestión operativa.
- **Multi-empresa** — Eldra se mantiene en su propia base. No hay integración activa.
- **App nativa** — todo es web. GitHub Pages + Airtable API es suficiente para esta escala.

---

## 8. Resumen de lo que cambia

| Área | v1 (actual) | v2 (nuevo) |
|------|-------------|------------|
| Tablas activas | 12 (incluyendo Eldra, Recetas, Materiales) | 8 |
| Campos totales | ~350 | ~90 |
| ID de ficha taller | id_produccion (por mueble) | id_pedido (por pedido, todos los muebles comparten) |
| Producción vs Pedidos | 2 tablas sincronizadas a mano | 1 Pedido + Items_Pedido simples |
| Estados del taller | "En Produ", "En produ - herreria", etc. | Nuevo / En herrería / En carpintería / En pintura / Para entregar / Entregado / Rehacer / Cancelado |
| Precio de catálogo | Variantes con fórmulas de costeo complejas | Catalogo simple con precio_lista manual |
| Productos custom | Requiere crear variante ad-hoc | Precio libre en Pedido, sin entidad de catálogo |
| Envío e instalación | Campos de monto separados (sin uso) | Eliminados — todo en precio_pactado + notas_logistica |
| Fecha entrega | Fórmula seña+15 días fija | Fórmula seña+30+dias_adicionales (ajustable) |
| Resultados (Airtable) | 25+ fórmulas y lookups | Solo índice de meses — cálculos en el dashboard |
| Frontend del taller | Vista Airtable en desktop | Interfaz mobile dedicada |
| Dashboard | HTML con errores de cálculo | HTML reconstruido, datos correctos |
| Carga de movimientos | Manual o vía Módulo 1 | Igual, pero con mapa de IDs actualizado |
| Alertas | Manual (Lisandro revis