# Decisiones técnicas

## Flutter para Android e iOS

Un único código base para ambas plataformas, relevante porque los dispositivos de almacén no son necesariamente homogéneos y la app debía instalarse en terminales de distintos operadores.

## Hive como persistencia local, con escritura inmediata por escaneo

En vez de mantener el registro de escaneo solo en memoria (y perderlo si la app se cierra o el dispositivo se queda sin batería a mitad de una recepción), cada escaneo se persiste de inmediato en Hive. Esto hace que la app sea offline-first "por diseño" — no necesita saber si hay red para poder seguir funcionando, porque el camino normal ya es local.

## Sin librería de detección de conectividad

El proyecto no usa `connectivity_plus` ni similar. La detección de "sin conexión" es reactiva: ocurre cuando falla la llamada de red al cerrar la orden (capturando `SocketException`), no antes. Es una decisión razonable dado que la única operación que requiere red es el cierre de la orden — no hay necesidad de monitorear conectividad de forma continua si solo un paso del flujo la usa.

## Honeywell Scanner SDK en vez de escaneo por cámara

El proyecto integra el SDK de Honeywell en vez de una librería de escaneo por cámara (como `mobile_scanner` o ML Kit). Esto indica que los dispositivos objetivo son terminales Honeywell con lector físico integrado — una decisión que viene dada por el hardware que usa el cliente en el almacén, no una preferencia de librería.

## BLoC/Cubit en las pantallas con más lógica, `setState` en las más simples

No es una decisión documentada, pero el patrón observable es: donde hay más complejidad de estado (autenticación con selección de empresa, detalle de orden con escaneo y reconciliación offline), se usa Cubit; en la pantalla de búsqueda de órdenes, más simple, se maneja con `setState`. Lo señalo como observación honesta del código, no como un estándar aplicado de forma consistente en todo el proyecto.

## Freezed solo en los modelos de autenticación

El resto de modelos de la app usan clases planas con `fromJson`/`toJson` manuales. Freezed se aplicó únicamente en `LoginResponse`/`LoginParams` — no hay evidencia de que se haya extendido al resto del proyecto.
