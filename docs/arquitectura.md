# Arquitectura

## Organización del código

Arquitectura **feature-first**: cada funcionalidad vive en su propia carpeta bajo `lib/features/`, con separación interna entre `data/` (modelos + servicios) y `presentation/` (pantallas, widgets y manejo de estado).

```
lib/
├── features/
│   ├── authentication/     (login, selección de empresa)
│   ├── pending_purchase_orders/   (búsqueda y listado de órdenes)
│   ├── purchase_order_detail/     (detalle, escaneo, registro)
│   ├── register_purchase_order/   (registro de código de barras nuevo)
│   ├── menu/                      (navegación / drawer)
│   └── log/                       (logging remoto de errores)
├── shared/          (constantes, estilos, widgets comunes)
└── utils/services/hive/   (capas de persistencia local)
```

Cada feature con lógica de red sigue el mismo patrón: un archivo `*_dio.dart` como capa de servicio (llamadas HTTP + manejo de errores), modelos de request/response, y una pantalla o cubit que orquesta la llamada.

## Manejo de estado

Mezcla de dos enfoques, verificado directamente en el código:

- **BLoC/Cubit** (`flutter_bloc`) en autenticación (`LoginCubit`, `CompaniesCubit`), en el detalle de orden de compra (`PurchaseOrderDetailCubit`) y en la navegación (`NavDrawerBloc`, con eventos y estados basados en `Equatable`).
- **`StatefulWidget` con llamadas directas a los servicios**, sin Cubit, en la pantalla de búsqueda de órdenes pendientes (`ConsultPurchaseOrdersScreen`) — el estado de carga/error se maneja con `setState` en vez de un Cubit dedicado.

Es una inconsistencia real del proyecto, no una decisión deliberada documentada — la pantalla de detalle (con la lógica offline más compleja) sí usa Cubit; la de búsqueda, más simple, no.

## Modelos

- **Freezed** (`login_response.dart`, `login_params.dart`) para los modelos de autenticación, con generación de `copyWith`/igualdad/serialización.
- El resto de modelos (`PendingPurchaseResponse`, `DetailPurchaseOrders`, `CompaniesResponse`, etc.) son clases planas con `fromJson`/`toJson` escritos a mano, sin Freezed.
- **Hive**: `DetailPurchaseOrders`, `PurchaseOrderEntry` y `Serie` son `HiveObject` con `@HiveType`/`@HiveField` y adapters generados (`.g.dart`), registrados en `main.dart` al iniciar la app.

## Capa de red

Un único `Dio` se crea en `main.dart` y se inyecta manualmente a cada servicio (sin un contenedor de inyección de dependencias como `get_it` o `injectable`) — la mayoría de las pantallas reciben sus dependencias por constructor. Cada servicio agrega el header `Authorization: Bearer <token>` antes de cada llamada, usando el token guardado en Hive tras el login.

## Persistencia local (Hive)

Cajas (`Box`) separadas para: token de sesión, usuario, empresa seleccionada, maestro de códigos de barras, y la entidad principal `PurchaseOrderEntry` (la orden de compra con su detalle, mientras se está registrando). El detalle completo del ciclo de vida de estos datos está en [`desarrollo.md`](desarrollo.md).

## Por qué esta arquitectura (y qué **no** es)

- No hay un patrón formal de Clean Architecture con capas de dominio/casos de uso — es feature-first con separación data/presentation, sin una capa de dominio explícita.
- No hay inyección de dependencias con un framework — las dependencias se construyen e inyectan manualmente.
- No hay detección proactiva de conectividad (sin `connectivity_plus`) — el manejo offline se apoya en que el escaneo escribe siempre en local, y el error de red solo se maneja en el paso de envío final.
