# Clase 3 — clave de casos

Uso del docente. **No repartir antes de la práctica.**

Trece eventos. La tabla de decisión correcta es esta.

| Evento | Qué trae | Respuesta HTTP | Efecto |
| --- | --- | --- | --- |
| `evt_2001` | Aprobado, Ana, `SEM-PRO`, 24500 ARS, monto exacto | `200` | Alta de acceso: `c-101 c-102 c-201 c-202`, vence `2026-06-08` |
| `evt_2002` | Aprobado, Bruno, `SEM-BAS`, 12000 ARS | `200` | Alta de acceso: `c-101 c-102`, vence `2026-04-09` |
| `evt_2003` | **Rechazado** por fondos insuficientes | `200` | Ningún acceso. Aviso al equipo comercial. |
| `evt_2004` | **Pendiente** | `200` | Ningún acceso. Se registra y se espera el evento definitivo. |
| `evt_2005` | Aprobado, correo **que no está en el padrón** | `200` | Ningún acceso. Cola de revisión. Devolver 404 acá es un error: el pago existe y el problema es nuestro. |
| `evt_2006` | Aprobado, usuario **suspendido** | `200` | Ningún acceso. Cola de revisión con motivo. |
| `evt_2007` | Aprobado por 12000, pero el plan `SEM-PRO` vale 24500 | `200` | Ningún acceso. Cola de revisión, motivo `monto_no_coincide`. |
| `evt_2008` | **Repite `pay_9001`** de `evt_2001` | `200` | Sin efecto. La respuesta es la misma que la primera vez. |
| `evt_2009` | Aprobado, `plan_code: SEM-GOLD`, **que no existe** en el catálogo | `200` | Ningún acceso. Cola de revisión. |
| `evt_2010` | Aprobado, 55 **USD** | `200` | Ningún acceso. Comparar 55 contra 48000 da falso, pero por la razón equivocada: hay que cortar por moneda antes de comparar importes. |
| `evt_2011` | **Falta** `data.payment_id` | `400` | Nada. El cuerpo no cumple el contrato. |
| `evt_2012` | Secreto incorrecto en la cabecera | `401` | Nada. Ni siquiera se lee el cuerpo. |
| `evt_2013` | Aprobado por 15000 para un plan de 12000: **paga de más** | `200` | Caso abierto a propósito. Ver abajo. |

Resultado: **2 accesos otorgados**, 6 a cola de revisión, 2 sin efecto por
estado del pago, 1 duplicado absorbido, 1 rechazado con `400`, 1 con `401`.

## Los tres casos que hay que discutir en el aula

**`evt_2013`, el que paga de más.** La regla ingenua es `amount === precio` y
manda a revisión. La regla razonable es `amount >= precio` y otorga. Ninguna de
las dos es obviamente correcta y depende del negocio. Lo que no puede pasar es
que la decisión quede implícita en un operador de comparación que nadie
discutió.

**`evt_2005`, el usuario que no existe.** Tentación: responder `404`. Está mal.
El emisor del webhook no tiene nada que arreglar y va a reintentar el envío
hasta agotarse. Un `404` acá genera reintentos infinitos por un problema que es
nuestro. Se responde `200` y el caso va a una cola.

**`evt_2010`, la moneda.** Si el flujo compara importes sin mirar la moneda, en
algún momento va a otorgar un acceso de 48000 pesos por 55 dólares, o al revés.
El punto pedagógico es que una comparación numérica correcta puede ser una
decisión incorrecta.

## Cálculo de vencimientos

`vence = fecha del evento + duracion_dias del plan`

- `evt_2001`: 2026-03-10 + 90 = **2026-06-08**
- `evt_2002`: 2026-03-10 + 30 = **2026-04-09**

Sirve para verificar de un vistazo si alguien sumó los días sobre la fecha de
ejecución en vez de sobre la del evento. La diferencia no se nota en clase y sí
se nota en producción.

## Alta de acceso esperada

Cuerpo que el flujo tiene que enviar al servicio de accesos:

```json
{
  "user_id": "usr_0031",
  "contenidos": ["c-101", "c-102", "c-201", "c-202"],
  "vence": "2026-06-08",
  "payment_id": "pay_9001"
}
```

`payment_id` viaja para que el servicio de accesos también pueda descartar
duplicados. La idempotencia se defiende en las dos puntas, no en una sola.

## Cómo disparar los eventos

Cada elemento del archivo trae las cabeceras y el cuerpo por separado, para que
la cabecera del secreto llegue como cabecera y no adentro del JSON.

Con `curl`, uno por uno:

```bash
curl -X POST "$URL_WEBHOOK" \
  -H 'content-type: application/json' \
  -H 'x-webhook-secret: s3m1n4r10-auto-2026' \
  -d '{ "id": "evt_2001", "type": "payment.succeeded", ... }'
```

Desde n8n, con un flujo aparte que hace de emisor: disparador manual, un nodo de
código que devuelve el contenido del archivo como items, y un nodo HTTP que
recorre los items armando cabeceras y cuerpo desde cada uno. Conviene tener ese
flujo emisor armado de antemano y compartido con el aula: dispararlos a mano de
a uno consume la mitad de la práctica.

## Sobre el secreto compartido

`x-webhook-secret: s3m1n4r10-auto-2026`

Es la versión simple de la autenticación. En producción se usa una firma HMAC
sobre el cuerpo, que además prueba que el contenido no fue alterado. La
diferencia se menciona en clase y se implementa en la unidad 4, cuando ya
tengamos el nodo de código encima de la mesa.

## Falla inducida, si sobra tiempo

Cambiar en el catálogo el precio de `SEM-PRO` de 24500 a 24000 sin avisar.
`evt_2001` deja de otorgar acceso y cae en la cola de revisión, sin ningún
error visible en el registro de ejecución. Es el ejemplo de que un flujo puede
estar perfecto y el resultado ser incorrecto porque cambió un dato de
referencia.
