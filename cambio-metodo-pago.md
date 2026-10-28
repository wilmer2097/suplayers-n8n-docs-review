# Cambio de método de pago — guía para n8n

Actualizada el 28 de octubre de 2026. Referencias ficticias. Sustituir TU_PROYECTO y los IDs por valores de la instalación.

[Explorar el flujo interactivo](https://wilmer2097.github.io/suplayers-n8n-docs-review/#tarjeta_inicio)

# Cambiar método de pago sin comprar — 26/10/2026

Actualización del 27/10/2026: el equipo indicó `POST /webhook/Actualizar-metodo-v1` para el formulario de Empresa. El mismo webhook recibe `accion_solicitada: cambiar_metodo_pago` y, para consultar un intento sin repetir la actualización, `accion_solicitada: consultar_cambio_metodo_pago`. La consulta lleva la misma clave y no lleva token. n8n las mapea respectivamente a `preparar` y `consultar`, validando primero el JWT del comprador. Las firmas y acciones de la función permanecen iguales. La prueba integral con Culqi test queda pendiente.

Función instalada: `public.sp_metodo_pago_cambiar`. Se usa desde un workflow nuevo de n8n. No modifica plan, fechas, precio ni consentimiento comercial de renovación. No crea filas en culqi_transacciones. No llama a Culqi ni cobra por sí sola.

## Contrato y acceso

POST https://TU_PROYECTO.supabase.co/rest/v1/rpc/sp_metodo_pago_cambiar

Headers: Content-Type: application/json, apikey del proyecto y Authorization: Bearer JWT_TECNICO_N8N. La sesión técnica debe tener pagos_gestionar. Antes de llamar, n8n valida el JWT recibido del usuario con Supabase Auth y usa el UUID verificado como p_comprador_uid; no acepta el UUID del body como identidad.

Una sola función, cinco acciones: preparar, consultar, confirmar, rechazar y desconocido.

| Parámetro | Origen |
| --- | --- |
| p_accion | Lo decide n8n según la etapa. |
| p_comprador_uid | ID devuelto al validar el JWT del comprador en Auth. |
| p_entidad_tipo | empresa o proveedor, de la selección del frontend; la función valida la vinculación. |
| p_entidad_id | Empresa/proveedor de la selección; no identifica al comprador. |
| p_clave_idempotencia | Referencia estable generada para el cambio. No cambiar por un timeout. |
| p_cambio_id | cambio_id de la preparación; requerido para confirmar/rechazar/desconocido. |
| p_consentimientos | Aceptación explícita del frontend, requerida al preparar. |
| p_resultado | Datos que n8n verificó en Culqi; no datos libres del navegador. |

La opción es para un operador activo super_admin, vinculado a la entidad, que sea el pagador designado de una asignación activa con renovación automática y tarjeta/cliente ya registrados. Se permite resolver el método de una asignación vencida aún activa, siempre sin cobro pendiente. No se cambian permisos para hacer pasar la validación.

## 1. Preparar

Frontend envía al nuevo webhook el JWT, token nuevo de Culqi, entidad, clave y consentimiento. El token solo se usa en n8n/Culqi: no se envía a la función ni se almacena en la tabla de seguimiento. No exponer tokens en logs del workflow.

Ejemplo parcial del frontend:

```json
{
  "accion_solicitada": "cambiar_metodo_pago",
  "entidad_tipo": "empresa",
  "entidad_id": 100,
  "token": "tkn_test_EJEMPLO_NUEVO",
  "clave_idempotencia": "cambio-metodo-EJEMPLO-001",
  "consentimientos": {
    "version": "cambio-metodo-v1",
    "usar_para_cobros_futuros": true
  }
}
```

La versión identifica el texto realmente mostrado al usuario; no inventar aceptación.

Llamada de n8n:

```json
{
  "p_accion": "preparar",
  "p_comprador_uid": "UUID_VERIFICADO_EN_AUTH",
  "p_entidad_tipo": "empresa",
  "p_entidad_id": 100,
  "p_clave_idempotencia": "cambio-metodo-EJEMPLO-001",
  "p_consentimientos": {
    "version": "cambio-metodo-v1",
    "usar_para_cobros_futuros": true
  }
}
```

Respuesta parcial ilustrativa:

```json
{
  "status": true,
  "cambio_id": "11111111-2222-4333-8444-555555555555",
  "clave_idempotencia": "cambio-metodo-EJEMPLO-001",
  "estado_cambio": "preparado",
  "autorizar_actualizacion": true,
  "reserva_hasta": "2026-10-26T17:10:00-05:00",
  "culqi_customer_id": "cus_test_EJEMPLO",
  "culqi_card_id": "crd_test_EJEMPLO"
}
```

Conservar cambio_id y clave durablemente antes del PATCH. Solo autorizar_actualizacion:true permite iniciar UNA actualización antes de reserva_hasta (10 minutos). Una repetición devuelve el estado y autorizar_actualizacion:false; no repetir el PATCH. Un timeout de preparar tampoco autoriza nada: consultar con la misma clave y conciliar si se perdió la autorización.

## 2. Actualizar en Culqi

Usar culqi_card_id en PATCH /v2/cards/{id} y el token nuevo como token_id, con la clave privada en credenciales de n8n. Este workflow actualiza el objeto existente: no crea otro Customer ni otro Card. Verificar con Culqi que la tarjeta está actualizada y que pertenece al culqi_customer_id recibido. Conservar marca y últimos cuatro verificados.

Referencia oficial: https://apidocs.culqi.com/#tag/Tarjetas/operation/actualizar-tarjeta

No hay nodo de cargo en este workflow. No usar sp_pago_iniciar, sp_pago_metodo_registrar ni sp_pago_resultado para esta operación independiente.

## 3. Confirmar y guardar resumen

```json
{
  "p_accion": "confirmar",
  "p_comprador_uid": "UUID_VERIFICADO_EN_AUTH",
  "p_entidad_tipo": "empresa",
  "p_entidad_id": 100,
  "p_clave_idempotencia": "cambio-metodo-EJEMPLO-001",
  "p_cambio_id": "11111111-2222-4333-8444-555555555555",
  "p_resultado": {
    "actualizacion_verificada": true,
    "customer_id": "cus_test_EJEMPLO",
    "card_id": "crd_test_EJEMPLO",
    "marca": "Visa",
    "ultimos_cuatro": "1113"
  }
}
```

Cliente y tarjeta deben coincidir con la preparación. Guardar una tarjeta con un ID diferente no está permitido en este flujo de actualización. El resumen se actualiza en operador; no se cambian los IDs del método ni los contratos de planes. La confirmación verificada puede registrarse después del vencimiento de la reserva. Repetir la misma confirmación devuelve el mismo resultado; un resultado distinto se rechaza.

Solo `estado_cambio:completado` significa actualización terminada. `status:true` por sí solo significa que se procesó la llamada, no que Culqi actualizó la tarjeta.

Respuesta pública que n8n puede enviar al frontend:

```json
{
  "status": true,
  "respuesta": "Tu método de pago fue actualizado.",
  "tarjeta": {"marca": "Visa", "ultimos_cuatro": "1113"}
}
```

No reenviar la respuesta técnica completa ni referencias cus/crd al navegador.

## 4. Timeout, consulta o rechazo

Usar los mismos datos de comprador, entidad, clave y cambio_id del intento.

- `p_accion:consultar`: devuelve el estado; basta la clave, cambio_id es opcional. No autoriza otro PATCH.
- `p_accion:desconocido`: deja el intento por_conciliar. Usar si no se sabe si Culqi actualizó. No repetir la actualización a ciegas.
- `p_accion:rechazar`, con `p_resultado:{"sin_actualizacion_verificada":true}`: solo si Culqi no cambió la tarjeta, o la ejecución terminó antes de enviar el PATCH y ya no puede enviarlo después.

El vencimiento de la reserva no prueba ausencia de actualización ni libera automáticamente el bloqueo. Consultar Culqi y luego confirmar o rechazar según evidencia. No reenviar el token consumido. Si Culqi actualizó pero falló el guardado en Supabase, volver a confirmar el mismo cambio tras verificar, sin otro PATCH.

## Bloqueo durante el cambio

La tabla privada metodo_pago_cambio registra la operación y su consentimiento. RLS habilitado, sin acceso directo para anon/authenticated/service_role. Solo la función técnica y helpers privados operan sobre ella.

Mientras existe un cambio preparado/por_conciliar, un trigger impide crear reservas financieras pendientes para la misma entidad. Las funciones de compra pueden devolver un error SQL METHOD_CHANGE_PENDING; Renovaciones puede envolverlo en una incidencia. n8n no debe cobrar cuando la reserva no fue autorizada. Las reservas existentes impiden preparar un cambio de tarjeta. Se comparte el bloqueo por entidad del flujo de pagos.

## Verificación y límites

Pruebas PostgreSQL 17 local con tablas mínimas y datos sintéticos: Empresa/Proveedor, permiso técnico, consentimiento, pagador automático, bloqueo por pago pendiente, repetición, nueva clave bloqueada, guardia de cargos, UUID incorrecto, reserva vencida, confirmación tardía/idempotente, cierre sin cambio y ausencia de modificaciones a planes/cargos. Fixtures de pruebas revertidos y servidor detenido. No prueba todos los triggers reales ni concurrencia de carga.

Instalación remota verificada con entradas inválidas, rechazo de identidad no autorizada, ACL y tabla privada sin operaciones creadas. Asesor: avisos esperados de tabla RLS sin políticas (deny-by-default y sin grants) y función pública SECURITY DEFINER autenticada con guarda interna. Otros hallazgos existentes no se modificaron. Referencia: https://supabase.com/docs/guides/database/database-linter .

Al instalar la función el 26/10 no se modificaron frontend ni workflows n8n. Desde el 27/10 el formulario de Empresa está configurado con el webhook indicado arriba. No hubo tokenización, PATCH real de Culqi ni cargos durante nuestra verificación. Queda pendiente la prueba integrada con Culqi test. La función no puede verificar Culqi por sí sola: la sesión técnica de n8n debe aportar evidencia real. El bloqueo SQL no detiene llamadas a Culqi hechas fuera del flujo de reservas autorizado.


## Cómo responder al frontend según el estado

| Respuesta RPC | Acción de n8n | Respuesta pública |
|---|---|---|
| Preparado y autorizar_actualizacion true | Una actualización en Culqi dentro de la reserva, verificar y confirmar | Todavía no responder éxito |
| Preparado y autorizar_actualizacion false | Consultar ejecución original; no repetir PATCH | status false y aviso pendiente |
| Completado | Finalizar y refrescar resumen en frontend | status true, respuesta y tarjeta |
| Rechazado | Intento cerrado sin actualización | status false y respuesta |
| Por conciliar | Verificar en Culqi; confirmar o rechazar según evidencia | status false y aviso pendiente; no informar rechazo definitivo |
| status false en preparar | No actualizar en Culqi; devolver validación | status false y respuesta |
| status false en confirmar | Corregir/conciliar el mismo intento, sin otro PATCH | Avisar que se está verificando |

Un error HTTP sin JSON no prueba que no hubo actualización. No interpretar un fallo del guardado como rechazo de Culqi. No reenviar respuestas técnicas completas al comprador.
