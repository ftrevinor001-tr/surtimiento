# Surtimiento — Guía de migración y actualización

Esta guía te lleva de tu archivo actual (214 MB, `.xlsb`) a un esquema de **dos archivos ligeros**:

- **`Surtimiento_Motor.xlsx`** → datos crudos. Lo maneja 1 analista. Aquí se pega/actualiza la información.
- **`Surtimiento_Comprador.xlsx`** → archivo ligero (≈3 MB) que abren los compradores. Calcula el *Sugerido* en vivo.

---

## 1. Por qué tu archivo pesa 214 MB (diagnóstico)

| Causa | Detalle |
|---|---|
| Cada día de venta era una **columna nueva** | La hoja `Datos` llegaba a la columna **OG (≈366 columnas diarias)**, todas con encabezado "UNI MIN". |
| Esa matriz estaba **repetida en 5 hojas** | `Datos`, `Dev`, `Insumos`, `Ajustes Mab`, `Traspasos` — cada una ~4.25 millones de números (113 MB c/u). |
| **Más de 8.5 millones de fórmulas vivas** | Las fórmulas se extendían hasta la fila ~43,766 aunque solo usas ~21,884 productos: medio archivo calculaba sobre celdas vacías. |
| `calcChain` de 127 MB | Consecuencia de lo anterior: Excel reordena millones de celdas en cada cambio. |

**La idea de fondo del cambio:** pasar los días de **columnas → filas** (formato "largo"). Así Excel y Power Query los procesan rápido, y agregar un día nuevo es *agregar filas*, no inflar el ancho.

---

## 2. Estructura nueva del MOTOR

| Hoja | Una fila por… | Columnas clave |
|---|---|---|
| `Catalogo` | clave de material (única) | ID, descripción, marca, grupo, comprador, empaque… |
| `VentasDiarias` | clave + fecha + sucursal | ID, FECHA, SUCURSAL, VENTA SIN DEV |
| `Movimientos` | clave + fecha + tipo | ID, FECHA, TIPO (*Dev/Insumo/Traspaso/AjusteMabasa*), SUCURSAL, CANTIDAD |
| `Parametros` | clave de material | COSTO UC, TIEMP ENTR, PEDID MAX, existencias, BO… |

> Las 4 hojas idénticas viejas (Dev/Insumos/Ajustes/Traspasos) se **fusionan en `Movimientos`**, distinguidas por la columna `TIPO`. Eso elimina ~340 MB de un golpe.

---

## 3. Migración inicial (una sola vez)

### 3.1 Convertir el .xlsb a .xlsx
1. Abre tu `Surtimiento__vacios___11_06_26__Nvo.xlsb` en Excel.
2. Quita la protección de cada hoja con la contraseña **3108** (Revisar → Desproteger hoja).
3. *Archivo → Guardar como → Libro de Excel (.xlsx)*. Esto solo es para extraer datos; será pesado pero temporal.

### 3.2 Llenar el Catálogo
- En tu hoja `Datos`, copia las columnas A–E + estatus + empaque + comprador/jefe (columnas A a AB aprox.).
- Pégalas en `Motor → Catalogo` debajo de los encabezados (pega **solo valores**: clic derecho → Pegado especial → Valores).
- Borra la fila de ejemplo amarilla.

### 3.3 Convertir las ventas diarias (columnas → filas)
Tus ventas viven en `Datos`, columnas **AF a OG** (una por día). Para volverlas filas:

1. En el .xlsx temporal, selecciona el bloque ID + las columnas diarias.
2. Excel: pestaña **Datos → Obtener datos → Desde otras fuentes → Desde tabla/rango**.
3. En el editor de Power Query, selecciona la columna ID → **Transformar → Anular dinamización de otras columnas** (Unpivot). Esto convierte las 366 columnas en 2: *Atributo* (la fecha) y *Valor* (la venta).
4. Renombra: Atributo → `FECHA`, Valor → `VENTA SIN DEV`. Agrega/ajusta `SUCURSAL`.
5. **Cerrar y cargar en**… elige pegar el resultado en `Motor → VentasDiarias`.

Repite la misma operación Unpivot para `Dev`, `Insumos`, `Ajustes Mab` y `Traspasos`, mandando todo a `Movimientos` y poniendo el `TIPO` correspondiente en cada bloque.

### 3.4 Parámetros
- De `Datos` copia COSTO UC, TIEMP ENTR, PEDID MAX, existencias, BO → pégalos (valores) en `Motor → Parametros`.

Guarda el Motor. **Ahora pesa una fracción del original** porque ya no hay fórmulas ni matrices repetidas.

---

## 4. Conectar el Comprador con el Motor (Power Query)

En **`Surtimiento_Comprador.xlsx`**, las hojas `AGG_Ventas`, `AGG_Movs`, `Catalogo` y `Parametros` se llenan automáticamente desde el Motor. Crea estas consultas:

**Datos → Obtener datos → Desde un archivo → Desde libro de Excel** → elige `Surtimiento_Motor.xlsx`.

Para los agregados de venta, en el editor pega esto en una consulta en blanco (*Nueva consulta → Consulta en blanco → Editor avanzado*), ajustando la ruta:

```
let
    Origen = Excel.Workbook(File.Contents("C:\Surtido\Surtimiento_Motor.xlsx"), null, true),
    Ventas = Origen{[Item="VentasDiarias",Kind="Sheet"]}[Data],
    Encab = Table.PromoteHeaders(Ventas, [PromoteAllScalars=true]),
    Tipos = Table.TransformColumnTypes(Encab,{{"ID ARTICULO", type text}, {"FECHA", type date}, {"VENTA SIN DEV (uni min)", type number}}),
    // últimos 12 meses
    Corte = Date.AddMonths(Date.From(DateTime.LocalNow()), -12),
    Filtrado = Table.SelectRows(Tipos, each [FECHA] >= Corte),
    Agrupado = Table.Group(Filtrado, {"ID ARTICULO"}, {
        {"VtaTotal_12m", each List.Sum([#"VENTA SIN DEV (uni min)"]), type number},
        {"VtaProm_Mensual", each List.Sum([#"VENTA SIN DEV (uni min)"]) / 12, type number}
    })
in
    Agrupado
```

Carga el resultado en la hoja `AGG_Ventas`. Haz consultas equivalentes para `AGG_Movs` (agrupando por `ID ARTICULO` y `TIPO`, luego pivotea TIPO a columnas Dev/Insumo/Traspaso/AjusteMabasa), `Catalogo` y `Parametros` (estas dos son carga directa de la hoja, sin agrupar).

> Si prefieres, en lugar de Power Query puedes pegar manualmente los agregados; pero Power Query es lo que mantiene el archivo ligero y confiable.

---

## 5. Cómo se calcula el Sugerido (en la hoja `Sugerido`)

El cálculo replica tu lógica de `Niveles` de forma transparente:

```
Pronóstico mensual = Venta promedio mensual neta × (1 + % crecimiento)
Demanda en tiempo de entrega = Pronóstico mensual × (Tiempo entrega días / 30)
Sugerido bruto = Pronóstico mensual × Meses objetivo
               + Demanda en tiempo de entrega
               − Existencia total (Sanver + Más Ferre)
               − Backorder (si CONSIDERAR BO = "Si")
Sugerido final = redondeado hacia arriba al múltiplo de empaque (CEILING)
Inversión = Sugerido final × Costo UC
Estado = "COMPRAR" si Sugerido final > 0, si no "OK"
```

Los **supuestos ajustables** (meses objetivo, periodo de análisis, % crecimiento, considerar BO) están en la hoja `Supuestos`, en celdas amarillas. Cámbialos y todo recalcula al instante.

---

## 6. Actualización diaria (rutina del analista)

1. Abre `Surtimiento_Motor.xlsx`.
2. En `VentasDiarias`, **agrega filas** con las ventas del día (ID, FECHA, SUCURSAL, cantidad). Nunca columnas.
3. En `Movimientos`, agrega devoluciones/insumos/traspasos/ajustes del día con su `TIPO`.
4. Actualiza existencias/BO en `Parametros` si cambiaron.
5. Guarda y cierra el Motor.
6. Avisa a los compradores: al abrir su archivo, **Datos → Actualizar todo** (Ctrl+Alt+F5) y verán el sugerido del día.

---

## 7. Sugerencias sobre el cálculo (opcionales, para mayor confiabilidad)

1. **Demanda neta de devoluciones:** hoy el sugerido usa venta promedio; conviene restar devoluciones reales del periodo para no sobre-comprar artículos con alta devolución.
2. **Estacionalidad:** en lugar de un promedio plano de 3 meses, pondera los mismos meses del año anterior (ej. comprar para diciembre con base en el diciembre pasado, no en octubre).
3. **Stock de seguridad por variabilidad:** añadir un colchón = factor × desviación estándar de la demanda, mayor para claves de venta irregular. Eso reduce faltantes sin inflar inventario en todo el catálogo.
4. **Excluir claves en LIQUIDACIÓN o status de compra cerrado** del sugerido, para que no aparezcan como "COMPRAR".
5. **Tope por inversión/pedido máximo:** ya está el redondeo a empaque; puedes agregar un límite de inversión por comprador o por proveedor.

Dime cuáles de estas quieres y las integro directo en las fórmulas de la plantilla.
