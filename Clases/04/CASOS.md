# Clase 4 — clave de casos

Uso del docente. **No repartir antes de la práctica.**

Ocho cotizaciones. El resultado correcto es:

| Salida | Cantidad |
| --- | --- |
| Publicadas en el panel | **6** |
| Descartadas con motivo | **2** |
| Llamadas HTTP al destino | **1** |

Si hay seis llamadas en vez de una, faltó juntar los items antes del nodo HTTP.
Además de publicar seis paneles de una fila, gasta seis veces la cuota.

## El token: qué pedir y qué no

Los alumnos necesitan un **fine-grained personal access token** de GitHub, con
un permiso y nada más:

> Settings → Developer settings → Personal access tokens → Fine-grained tokens
> → Generate new token → **Account permissions → Gists → Read and write**

Es gratis, instantáneo y no pide tarjeta.

**Insistir en el permiso mínimo.** Un token con acceso a los repositorios
pegado en una instancia de n8n compartida es un problema de seguridad real, no
un ejercicio. El permiso de gists no toca ningún repositorio.

En n8n va como credencial genérica **Header Auth**:

| Campo | Valor |
| --- | --- |
| Name | `Authorization` |
| Value | `Bearer github_pat_...` |

El error más frecuente es **olvidarse el `Bearer ` adelante**, o copiar el token
con un espacio al final. Las dos cosas dan `401`.

## La demostración del límite

Los dos nodos de arriba llaman a `GET https://api.github.com/rate_limit`. Misma
URL, mismo método. Uno tiene la credencial seleccionada y el otro no.

| | `resources.core.limit` |
| --- | --- |
| Sin credencial | **60** por hora, **por dirección IP** |
| Con credencial | **5.000** por hora, por cuenta |

Son 83 veces más. El número sale en el cuerpo de la respuesta y también en la
cabecera `x-ratelimit-limit` de cualquier llamada a la API.

**Si el nodo sin credencial devuelve `403`**, el aula ya agotó entre todos el
límite anónimo de la IP del edificio. No es un problema: es la lección llegando
antes de tiempo, y conviene detenerse ahí treinta segundos. El nodo está
configurado con `onError: continueRegularOutput` justamente para que el flujo
siga y el nodo de comparación lo explique solo.

## Defectos plantados, fila por fila

| Casa | Defecto | Qué tiene que pasar |
| --- | --- | --- |
| `oficial` | — | Publica. Spread 40. |
| `blue` | — | Publica. Spread 20. |
| `bolsa` | — | Publica. Spread 6.4. Decimales, por si alguien redondea de más. |
| `contadoconliqui` | — | Publica. Spread 7.2. |
| `mayorista` | Importes como **texto**: `"1015.00"` | **Publica.** Se convierten. No es un error del proveedor: es su contrato. El que compara sin convertir lo descarta o calcula mal. |
| `cripto` | **Venta menor que compra** | Descartada, motivo `venta_menor_que_compra`. Numéricamente válido, comercialmente imposible. Ninguna validación de tipo lo agarra. |
| `tarjeta` | `compra: null` | **Publica**, con compra y spread en nulo. Esa casa no publica precio de compra: el nulo es legítimo. Es la lección de "ausente no es inválido", por tercera vez en el seminario. |
| `mep` | **Falta** `fechaActualizacion` | Descartada, motivo `fecha_ausente`. Discutible y por eso vale la pena: ver abajo. |

## Los dos casos para discutir

**`tarjeta` contra `mep`.** Las dos filas tienen un campo faltante y terminan
en lados opuestos. La diferencia no es técnica: un precio de compra que no
existe es información correcta, y una cotización sin fecha es información que
no se puede usar. Si alguien descarta `tarjeta`, no está equivocado en el
código: está equivocado en el criterio, y ahí es donde conviene detenerse.

**La fecha ausente se podría completar con la de ejecución.** Alguien lo va a
proponer. Es exactamente lo que hicimos en la clase 2 con la fila sin fecha del
evento. La diferencia: allá la fecha era metadato de un contacto, acá es el dato
que le dice al usuario si la cotización es de hoy o de la semana pasada.
Inventarla es mentir en pantalla.

## Panel esperado

```
mercado           compra    venta    spread
oficial           1020      1060     40
blue              1185      1205     20
bolsa             1142.5    1148.9   6.4
contadoconliqui   1160.2    1167.4   7.2
mayorista         1015      1018     3
tarjeta           null      1378     null
```

El cuerpo que sale al destino es **un solo objeto**, con esta forma:

```json
{
  "description": "Panel de cotizaciones USD — 2026-03-17",
  "public": false,
  "files": {
    "panel-cotizaciones.json": { "content": "{ ...el panel como texto... }" }
  }
}
```

Dos cosas que conviene señalar:

- **`content` es un texto**, no un objeto. Hay un `JSON.stringify` adentro de
  otro. Se les va a hacer raro y está bien que pregunten: el archivo que se
  crea es un archivo de texto que adentro tiene JSON.
- **`public: false`.** Un gist público es público de verdad, indexable. No hay
  nada sensible en cotizaciones del dólar, y aun así la costumbre correcta es
  que lo que publica un flujo no sea público por descuido.

La respuesta trae `html_url`: esa es la dirección que abren en el navegador.

## Clasificación de errores esperada

El nodo de aviso tiene que separar transitorio de permanente:

| Error | Clase | Qué dice el aviso |
| --- | --- | --- |
| `401 Bad credentials` | permanente | La credencial no sirve: ausente, mal escrita o vencida. |
| `403 API rate limit exceeded` | **transitorio** | Límite agotado. Esperar a que se reponga. |
| `403 Resource not accessible` | permanente | Autenticado, pero al token le falta el permiso. |
| `503 Service Unavailable` | transitorio | Es de ellos y se pasa solo. |
| `422 Validation Failed` | permanente | El cuerpo no cumple el contrato. |
| `ENOTFOUND` (sin código) | transitorio | Sin código HTTP se asume red, que suele volver. |

**Los dos `403` son el caso interesante.** El mismo código significa dos cosas
opuestas según el cuerpo de la respuesta: una se arregla esperando y la otra no
se arregla nunca. Es la prueba concreta de que **el código de estado solo no
alcanza**, y de que leer el cuerpo antes que el número no es una preferencia de
estilo.

Si alguien pregunta por qué GitHub no usa `429` para el límite: lo usa a veces,
y otras veces manda `403`. Los proveedores reales son así, y el flujo tiene que
sobrevivir a eso.

## Las tres fallas inducidas

Van al final de la práctica, en este orden.

**1. Romper un carácter del token.** Editar la credencial, sacar una letra,
ejecutar. Llega un `401`. Dos cosas para mirar en el registro: que el reintento
hizo tres intentos inútiles y esperó diez segundos de más, y que el aviso lo
clasificó como permanente y dice qué revisar. Es el argumento de por qué el
reintento no es gratis.

**2. Tiempo de espera de 1 ms.** En las opciones del nodo de publicación. El
pedido sale y no vuelve a tiempo. La pregunta para el aula: ¿se creó el panel
del otro lado o no? No se puede saber desde acá — y la forma de averiguarlo es
listar los gists de la cuenta, que es trabajo extra que existe solo porque no
hubo clave de idempotencia. Es el caso que justifica la clase 3, ahora del lado
del emisor.

**3. Un campo renombrado en la validación.** Cambiar `venta` por `precio`. El
flujo corre entero, **ningún nodo se pone en rojo**, se publica un panel con
seis cotizaciones con la venta en nulo, y la respuesta trae una dirección que se
abre perfectamente. Es el cierre conceptual de la clase: el manejo de errores no
atrapa los errores que no son excepciones.

Si sobra tiempo, dejar la tercera sin avisar qué cambiaron y que la encuentren
leyendo el registro de ejecuciones.

## Sobre los datos fijados

Los nodos de cotizaciones, de publicación y los dos de la demostración vienen
con `pinData`. El flujo corre completo sin red, que es lo que hace la clase
inmune a una caída de internet del aula.

Para trabajar en vivo hay que quitar el pin. Conviene hacerlo por partes: los
dos de la demostración en el paso 0 (son gratis y rápidos), y los del flujo
principal recién en el paso 5, cuando ya está todo armado, así las fallas
inducidas son reales y no simuladas.

La API de origen es pública, gratuita y sin clave. El destino es la API de
GitHub, que pide credencial para escribir. Que el origen no pida nada y el
destino sí es parte del contenido: leer algo público es gratis, dejar algo
escrito en el mundo no.
