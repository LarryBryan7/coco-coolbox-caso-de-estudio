# Desarrollo

Detalle técnico verificado directamente en el código del proyecto.

## Escaneo de código de barras

- Librería: **`honeywell_scanner`** (SDK del fabricante), no la cámara del teléfono. Confirma que los dispositivos usados son lectores/terminales Honeywell (hardware dedicado de almacén), no smartphones genéricos.
- Al iniciar el escaneo (`HoneywellScanner().startScanner()`), se configuran las propiedades del lector para aceptar formatos 1D y 2D (`ALL_1D_FORMATS`, `ALL_2D_FORMATS`), con ajustes específicos para Codabar y EAN-13.
- El resultado del escaneo llega vía callback (`onDecoded`), que devuelve el código leído a la pantalla que abrió el diálogo de escaneo.
- Mientras se escanea, se muestra una animación Lottie ("Escaneando, por favor espere...").
- Tras obtener el código, se busca en el **maestro de códigos de barras cacheado en Hive** (`getBarcodeLocal`) para identificar el producto, sin necesidad de red.

## Persistencia local con Hive

**Qué se almacena:**
- `PurchaseOrderEntry` — la orden de compra en curso (proveedor, fecha, almacén, y su lista de detalle).
- `DetailPurchaseOrders` — cada línea de producto de la orden (código, descripción, cantidad requerida, cantidad recibida, si es serializado).
- `Serie` — números de serie/IMEI registrados para productos serializados.
- Maestro de códigos de barras, token de sesión, usuario y empresa seleccionada (cajas separadas).

**Cuándo se escribe:**
- Al abrir el detalle de una orden por primera vez: se descarga desde la API y se guarda íntegra en Hive (`savePurchaseOrderEntry`).
- Al escanear cada producto (`registerProduct` → `updateOrderByCode`): se actualiza **solo en Hive** — incrementa `cantidadRecibida` para ese código, y agrega el número de serie si corresponde (evitando duplicar una serie ya registrada). No hay llamada de red en este paso.
- Al cerrar la orden exitosamente: se eliminan todas las cajas relacionadas a esa orden (`deleteAllPurchaseOrderEntry`).

**Reconciliación local ↔ servidor:**
Si el usuario vuelve a abrir una orden que ya tenía datos guardados localmente (`isRegistered = true`), la app pide el detalle actualizado al servidor y ejecuta `syncLocalWithServer`: compara cada línea local contra la versión del servidor por código de producto, y si la cantidad disponible del servidor cambió (por ejemplo, otro operador ya registró parte de esa orden), ajusta la cantidad local para no exceder lo realmente disponible. Es una reconciliación por campo, no un reemplazo total del registro local.

**Recuperación tras cierre inesperado:**
Si al reabrir la app hay una orden guardada en Hive sin cerrar, se muestra un diálogo ("Orden de compra pendiente de cierre... ¿Desea continuar el registro?") — visible en las capturas de este repositorio.

## Envío al servidor (cierre de orden)

Al presionar "Cerrar Orden de Compra": se filtran solo las líneas con `cantidadRecibida > 0`, se arma el payload con la orden completa (proveedor, guía, comentario, detalle filtrado) y se envía por `POST` con el token Bearer correspondiente.

Manejo de respuesta:
- `200` → éxito, se limpia Hive.
- `400` → error de validación estructurado (lista de mensajes por campo), se muestra al usuario sin lanzar excepción genérica.
- Excepción con `SocketException` de fondo → mensaje específico "No hay conexión a internet. Por favor, verifica tu conexión." (los datos permanecen en Hive para reintentar).
- Cualquier otro error → mensaje genérico, también preservando los datos locales.

## Logging remoto

Existe un servicio de logging (`LogService`) que registra errores hacia el backend con: servicio de origen, empresa, usuario, ID de dispositivo y mensaje — usado de forma consistente en los distintos servicios (`PurchaseOrderDetailService`, `LoginService`, `PendingPurchaseDio`) para dejar rastro de fallas de red o de servidor.

## Ejemplo ilustrativo de la reconciliación offline → online

> Ejemplo ficticio, con nombres de campos simplificados para explicar la mecánica — no corresponde a los contratos reales de la API de Coolbox.

```json
// Estado local (Hive) antes de reabrir la orden
{ "codigo": "SKU-0001", "cantidadRecibida": 4, "cantidadDisponible": 10 }

// Respuesta del servidor al reabrir (otro operador ya registró parte)
{ "codigo": "SKU-0001", "cantidadDisponible": 3 }

// Resultado tras la reconciliación local
{ "codigo": "SKU-0001", "cantidadRecibida": 3, "cantidadDisponible": 3 }
```
