# Tarjeta guardada y tarjeta nueva en el checkout

Actualizado: 22 de octubre de 2026. Alcance: frontend Empresa e integración con n8n.

## Qué cambia en el frontend

Al abrir el checkout se consulta el estado de facturación con la sesión autenticada del comprador. Si el backend permite usar su tarjeta guardada, se muestra la marca y los últimos cuatro dígitos. La opción se puede desmarcar para ingresar otra tarjeta.

El HTML recibe únicamente los indicadores necesarios y el resumen de tarjeta. Los identificadores privados del cliente y la tarjeta se resuelven en backend. Mientras se consulta se deshabilita continuar; si falla la consulta, se permite el camino de tarjeta nueva sin afirmar que no existe un método guardado.

La renovación automática sigue siendo una autorización separada: usar una tarjeta guardada no activa renovación por sí solo.

## Qué recibe n8n

Tarjeta guardada seleccionada:

```json
{
  "metodo_pago": "tarjeta_guardada",
  "usar_tarjeta_guardada": true,
  "tipo_cobro": "one_click"
}
```

No hay token nuevo. Mapear usar_tarjeta_guardada a p_usar_tarjeta_guardada en sp_pago_iniciar. La reserva debe incorporar las referencias autorizadas del comprador. No crear Customer/Card de nuevo ni omitir ese flag porque operador ya tenga tarjeta.

Tarjeta nueva con autorización para guardar y renovar:

```json
{
  "metodo_pago": "tarjeta_nueva",
  "usar_tarjeta_guardada": false,
  "tipo_cobro": "one_click",
  "token": "tkn_test_EJEMPLO",
  "guardar_tarjeta": true,
  "renovacion_automatica": true,
  "consentimientos": {
    "version": "TEXTO_MOSTRADO_AL_USUARIO",
    "guardar_tarjeta": true,
    "renovacion_automatica": true
  }
}
```

Son fragmentos del webhook; se conservan identidad verificada, entidad, plan, período, total, moneda y clave de idempotencia del contrato existente. No inventar consentimientos.

## Orden de las operaciones

1. Validar JWT del comprador contra Auth.
2. Llamar a sp_pago_iniciar con la sesión técnica y el método elegido.
3. Respetar accion: un cambio programado o registro de método no autoriza un cargo inmediato.
4. Para cobrar, exigir status:true, autorizar_cargo:true y reserva vigente. Conservar durablemente cqt_id y procesamiento_id originales.
5. Con tarjeta guardada, usar la referencia autorizada devuelta por la reserva.
6. Con tarjeta nueva autorizada para guardar o renovar, reutilizar el Customer existente cuando corresponda, crear/verificar Card y ejecutar sp_pago_metodo_registrar antes del cargo.
7. Si el registro del método falla, detenerse antes de cobrar.
8. Verificar el cargo en Culqi y llamar a sp_pago_resultado.
9. Comprobar por separado estado_pago y estado_aplicacion.

Para tarjeta nueva sin guardar ni renovar, mantener el camino de cargo único con token y sin método persistente.

## Pago exitoso pero plan no aplicado

RENEWAL_CARD_NOT_RECORDED indica que la aplicación no encontró un método consistente para la renovación solicitada. Que operador tenga una tarjeta no prueba que ese intento la haya vinculado correctamente.

No generar otra compra ni otro cargo. Verificar las referencias utilizadas en Culqi y corregir el intento mediante un procedimiento backend controlado. sp_pago_metodo_registrar no admite agregar el método después de que el pago quedó exitoso en el flujo actual.

Tras la corrección, repetir únicamente sp_pago_resultado con cqt_id, procesamiento_id original y los mismos datos del cargo verificado. Se reintenta la aplicación; no se vuelve a cobrar. Un timeout tampoco autoriza repetir un cargo.

## Estado de verificación

Cambio enviado al proyecto Empresa en FlutterFlow y exportación posterior comprobada. Pruebas JavaScript locales aprobadas para consulta de tarjeta, datos enmascarados, camino sin token, selección de otra tarjeta y fallo de consulta.

Pendiente: prueba integrada autenticada de ambos caminos con n8n y Culqi test. No se modificaron workflows n8n ni se recuperaron pagos previos en este cambio. No se acredita una publicación del frontend Proveedor.
