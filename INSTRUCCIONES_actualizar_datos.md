# Cómo actualizar los datos de la app de Surtimiento (cálculo exacto)

Sigue estos 3 pasos cada vez que quieras refrescar la información.

## Resumen de la arquitectura
- La APP (index.html) no cambia. Es lo que ven los compradores.
- Los DATOS (datos_surtimiento.js) es lo único que reemplazas.
- El cálculo del Pron (meses de inventario que usan los compradores) necesita
  las VENTAS DIARIAS, no mensuales. Por eso el archivo de datos se genera
  con el convertidor_v2.html.

---

## PASO 1 — Preparar el CSV de datos diarios

Tu archivo debe tener, EN ESTE ORDEN, estas columnas (encabezados exactos):

### Columnas de identificación e inventario (15)
CLAVE, DESCRIPCION, MARCA, GRUPO, COMPRADOR,
TIEMPO_ENTREGA_DIAS, MESES_OBJETIVO, PZS_EMPAQUE, COSTO_UNITARIO,
EXIST_SANVER, BO_SANVER, TRANSITO_SANVER,
EXIST_MASFERRE, BASE_MASFERRE, VENTA_MENSUAL_MASFERRE

### Ventas diarias (365)
DIA_001, DIA_002, ... DIA_365
  - DIA_001 = día MÁS RECIENTE
  - DIA_365 = hace un año
  (mismo orden que las columnas AF en adelante de tu hoja 'Datos sin dev')

### Opcionales (devoluciones, ajustes, insumos, traspasos)
DEV_001..DEV_365, AJU_001..AJU_365, INS_001..INS_365, TRA_001..TRA_365
  - Si no los incluyes, se toman como 0.

Una fila por clave. Guardar como CSV UTF-8.

CONSEJO: arma una hoja con Power Query que convierta tu export del sistema
a este formato, y reutilízala. Así solo pegas el export nuevo cada vez.

---

## PASO 2 — Convertir

1. Abre convertidor_v2.html (doble clic; corre en tu navegador, datos no salen de tu PC).
2. Paso 2 en pantalla: confirma "días transcurridos del mes" (el 14 de tu Excel,
   celda Z2) y la fecha de los datos.
3. Paso 3: arrastra tu CSV. Verás cuántas claves y días detectó.
4. Paso 4: pulsa "Generar datos_surtimiento.js" y descárgalo.

El convertidor pre-calcula la venta anual (pAnual) y los cortes de cada periodo,
así el archivo pesa ~2 MB en vez de 32 MB y el cálculo es exacto.

---

## PASO 3 — Publicar

1. En GitHub, sube el datos_surtimiento.js nuevo REEMPLAZANDO el viejo (mismo nombre).
2. Espera 1-2 minutos (GitHub Pages tarda en publicar).
3. Abre la app agregando ?v=3 al final de la URL para evitar el caché del navegador.
   Ejemplo: tusitio.github.io/surtimiento/?v=3
   (la próxima vez usa ?v=4, ?v=5, etc.)

---

## Cómo saber que cargó bien
El indicador de arriba a la derecha:
- VERDE con la fecha correcta  -> datos nuevos cargados bien.
- ROJO "formato ANTIGUO"        -> el .js es del convertidor viejo, regenéralo.
- ÁMBAR "MODO DEMO"             -> no encontró el .js (revisa el nombre/ubicación).

Para verificar el cálculo: busca la clave 1713 y revisa que el Pron
(columna MESES de SANVER) sea ~4.4. Si coincide con tu Excel, está validado.

## Importante
El datos_surtimiento.js VIEJO (el que tiene "hist") NO sirve con esta app.
Tiene que ser uno generado con convertidor_v2.html (que tiene "cortes" y "pAnual").
