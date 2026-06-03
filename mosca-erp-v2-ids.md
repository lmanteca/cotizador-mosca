# Mosca ERP v2 — IDs de referencia
**Generado:** 2026-06-01  
**Base:** `appBThGsxqCSljqH1`

---

## Tablas v2 (nuevas)

| Tabla | ID |
|-------|-----|
| Resultados_v2 | `tblSUklW69NGOLKEF` |
| Pedidos_v2 | `tblok3VLqWA3oQaPe` |
| Items_Pedido | `tblQ4Kq61LlZHFPNN` |
| Catalogo | `tbl8iau1GmyJ3faxl` |

## Tablas existentes (referencia)

| Tabla | ID |
|-------|-----|
| Ingresos | `tblS8mrLsVnt5yvSC` |
| Gastos | `tbll52naofFFppi2u` |
| Proveedores | `tblsa0E6xH6LMffmu` |
| Clientes | `tbl5tU70DznAaqNd6` |
| Resultados (v1) | `tbl08g5czTbEJTCOb` |

## Campos clave — Gastos

| Campo | ID |
|-------|-----|
| tipo_flujo (nuevo) | `flduA7RvBwo6KtINE` |

## Campos clave — Ingresos

| Campo | ID |
|-------|-----|
| tipo_flujo (ya existía) | `fldxVjhFYP4bZT1y3` |

---

## Registros Resultados_v2

| Mes | Record ID |
|-----|-----------|
| 2025-06 | `recLU4ZpDLMzCBuEF` |
| 2025-07 | `rec7YJn1cDFf6mMiK` |
| 2025-08 | `recECh69nCv89cBvo` |
| 2025-09 | `recQ8oYW93CIig21W` |
| 2025-10 | `recTn8sgBR4C4EFmx` |
| 2025-11 | `rec5qvpZ9heGRQQUd` |
| 2025-12 | `recBdO5Hsm05vlBbR` |
| 2026-01 | `rec6W2dYHINniE9oC` |
| 2026-02 | `recqKa1EYcWsxnDXm` |
| 2026-03 | `rech1dfv8p1ycHY0w` |
| 2026-04 | `recKG3XCIiEqBwnxR` |
| 2026-05 | `recf2nQfIA2f1Tf4a` |
| 2026-06 | `recgvkwGp6P6FVCyD` |
| 2026-07 | `rec2NSTNOMKMb4OdS` |
| 2026-08 | `recxd5GhJ2tMLOXZ1` |
| 2026-09 | `recYWMBN7F7NfOtcH` |
| 2026-10 | `rec2ra77EMtEZrqpx` |
| 2026-11 | `recJH92gOPH6OBDYc` |
| 2026-12 | `recgA7t8wRntHn7aK` |

---

## Campos clave — Items_Pedido

| Campo | ID | Notas |
|-------|-----|-------|
| catalogo (link → Catalogo) | `fldqBhPu2fsfduJHG` | creado 2026-06-02 |
| notas (notas comerciales) | `fldiznnQMxttDh4kr` | creado 2026-06-02, solo lectura en taller |
| notas_taller (alerta operativa) | existente | editable por Candela |

## Campos clave — Catalogo

| Campo | ID | Notas |
|-------|-----|-------|
| producto (primary field) | `fldBjvLNqI1cHER7U` | nombre del producto |
| precio_promocional | `fldBdeYPY8ksWXggi` | editable |
| precio_lista | `fldLJw0hXafjemMgJ` | fórmula (promo/0.8), no editable |
| imagen | `fldw6djxu31IBQDgU` | creado 2026-06-02 |

## Registros Catalogo (datos de prueba)

| Producto | Record ID |
|----------|-----------|
| Núcleo | `rec1ur4xkO898EcdL` |
| Mesa de comedor | `recX5ttkkYvDxH9rc` |
| Rack TV | `recsxPd1dEtTGKYeS` |
| Banco largo | `rec5m7fnrv6WWqH1k` |
| Estantería | `recAdrUDohcdqvSCV` |

## Pendiente manual en Airtable

- **Items_Pedido.nombre**: crear campo tipo Lookup → `catalogo` → `producto` (API no soporta crear lookups)
- **Items_Pedido.descripcion**: eliminar una vez el lookup `nombre` esté activo
- **Pedidos_v2.id_pedido**: ya tiene autoNumber (confirmado en schema)

## taller-interface.html — Estado actual (2026-06-02)

**Versión:** v1.6  
**URL:** `https://lmanteca.github.io/cotizador-