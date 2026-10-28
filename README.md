# Guía temporal de pagos n8n

Documentación interactiva para revisar la integración de pagos. Todos los ejemplos usan datos ficticios; el HTML no realiza cobros ni solicitudes a servicios externos.

Abrir index.html o consultar el despliegue de GitHub Pages. No incluir claves, JWT, resultados de pagos reales, datos de clientes, SQL del proyecto ni archivos del entorno en este repositorio.

## Publicación

GitHub Pages desde la rama main, carpeta raíz. El sitio es público y no implementa autenticación. Las indicaciones noindex y robots.txt solicitan no indexación; no son control de acceso.

## Retirar al terminar la revisión

En Settings > Pages > Build and deployment, desactivar la fuente de publicación (None) y guardar. Comprobar que la URL deja de servir la guía. Retirar el sitio no elimina automáticamente el contenido del repositorio ni copias de terceros.

## Alcance

47 casos de respuesta y catálogo técnico, con ejemplos parciales simulados. La guía documenta decisiones; no es un workflow importable ni sustituye validaciones del backend.

## Actualización del checkout — 22/10/2026

Ver [Tarjeta guardada y tarjeta nueva](TARJETA_GUARDADA.md): selección del método, mapeo de `usar_tarjeta_guardada`, registro antes del cargo y recuperación de pagos exitosos con aplicación fallida. Frontend Empresa actualizado en FlutterFlow; prueba integrada con n8n/Culqi test pendiente.

## Lectura de la guía

Cada función muestra qué enviar, de dónde sale cada parámetro, un ejemplo, la respuesta y el siguiente paso en lenguaje sencillo. Las condiciones de IF/Switch siguen disponibles como detalle desplegable. Renovaciones y Notificaciones tienen pestañas separadas; los avisos corresponden a 5, 3, 2 y 1 días.

## Cambio de método de pago — 28/10/2026

Nueva pestaña [Cambiar método de pago](https://wilmer2097.github.io/suplayers-n8n-docs-review/#tarjeta_inicio), con diez escenarios interactivos. Incluye preparación, actualización verificada, confirmación, consultas, rechazos y recuperación ante timeout. Ver [API, parámetros y cuerpos completos](cambio-metodo-pago.md). Este flujo no cobra ni cambia el plan o sus fechas; la prueba integral con Culqi test sigue pendiente.
