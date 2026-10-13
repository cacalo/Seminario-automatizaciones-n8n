# Clase 7 — clave de casos

Uso del docente. **No repartir antes de la práctica.**

El guion lo escribe el modelo y las fotos dependen de lo que haya en Unsplash
ese día, así que el video no es reproducible. Lo que sí es fijo: qué temas
pasan la curaduría, qué controla la validación del guion, cómo se comporta el
lazo de sondeo, cuánto cuesta una pieza y qué hace cada rama de la aprobación.

## Antes de la clase

- **Probar las cuatro credenciales con una cuenta propia** el día anterior, y
  revisar que el registro de Unsplash y de JSON2Video siga siendo gratuito y
  sin tarjeta.
- **Hacer el cliente OAuth de Google Cloud una vez, con capturas.** La consola
  cambia seguido de lugar las cosas, y es el paso que más tiempo se come.
- Tener a mano la **dirección de retorno** de la instancia:
  `https://<instancia>/rest/oauth2-credential/callback`. Es la misma para todos.
- **El resuelto viene con datos fijados** en varios nodos, entre ellos
  **Pedir el render** y **Pedir aprobación humana**. Sirve para mostrarlo sin
  gastar nada. Para correrlo de verdad hay que quitar el pin de **Pedir el
  render**: si no, **Consultar estado** pregunta por un `project` de otra
  cuenta con la clave del alumno.
- **Tener un video ya renderizado** para abrir la clase. Es la mejor
  motivación de la unidad y no cuesta créditos del aula.
- Cada alumno usa **sus propias claves**. Si se comparte una clave de Unsplash,
  el límite de 50 pedidos por hora se reparte entre todos y se agota en
  minutos. Si se comparte una de JSON2Video, los 600 segundos son del aula
  entera, no de cada uno.

## La curaduría: diez temas, seis afuera

| Tema | Resultado | Motivo |
| --- | --- | --- |
| `t-01` Por qué el cielo es celeste | descartado | `ya_publicado` |
| `t-02` Cómo duermen los delfines | **apto** | — |
| `t-03` por qué el cielo es celeste (otra vez) | descartado | `duplicado_de_t-01` |
| `t-04` (vacío) | descartado | `tema_vacio` |
| `t-05` El té de jengibre que cura la ansiedad | descartado | `promesa_de_salud_o_dinero` |
| `t-06` El secreto de Messi para no cansarse | **apto** | — (ver abajo) |
| `t-07` Agujero negro con pileta, manguera... | descartado | `demasiado_largo_para_30_segundos` |
| `t-08` Por qué los gatos ronronean | **apto** | — |
| `t-09` Cómo se forma un arcoíris | **apto** | — |
| `t-10` Bajá 5 kilos sin dieta | descartado | `promesa_de_salud_o_dinero` |

Cuatro aptos. El primero que sale es `t-02`.

**`t-03` es el interesante.** Está escrito distinto que `t-01`: minúscula al
principio y un espacio al final. Si alguien compara los textos tal cual, pasa.
La normalización es la de la clase 2, aplicada a otra cosa.

**El orden de los controles importa.** `t-03` se marca como duplicado de
`t-01` aunque `t-01` ya esté publicado, porque el duplicado se evalúa antes.
Si se invierte el orden, `t-03` aparece como apto: el tema que ya salió vuelve
a salir con otra mayúscula.

## `t-06`: la capa que sigue

El tema de Messi pasa la curaduría por código, a propósito. No hay forma
razonable de detectar "atribuye algo a una persona real" con una expresión
regular.

Lo puede atrapar el modelo: el prompt del sistema le pide `apto: false` en
ese caso, y la validación lo manda a **Guion para revisar** con el motivo. Lo
hace casi siempre, no siempre. Si no lo atrapa, llega a la persona que
aprueba, que es la última capa.

Para que el aula lo vea: cerrar `t-02` a mano (o ponerle `hecho: true`) y
ejecutar. La pregunta: **¿qué capa lo atrapó en tu flujo?** Las respuestas
van a ser distintas, y esa diferencia es el argumento de las capas.

## La validación del guion

Controles, en orden. El primero que falla es el motivo.

| Control | Motivo |
| --- | --- |
| El modelo dijo que no era apto | `el_modelo_lo_considero_no_apto: ...` |
| No son exactamente 4 escenas | `tiene_N_escenas_en_vez_de_4` |
| Alguna escena sin narración | `escena_sin_narracion` |
| Menos de 55 o más de 95 palabras | `N_palabras_fuera_de_55_a_95` |
| Alguna búsqueda vacía o en castellano | `busqueda_vacia_o_en_castellano` |
| Título vacío o de más de 70 caracteres | `titulo_vacio_o_largo` |

A 2,5 palabras por segundo, 55 a 95 palabras son 22 a 38 segundos de voz.

**Por qué las búsquedas en inglés.** Unsplash tiene casi todas sus
descripciones en inglés. "delfín bajo el agua" trae pocas fotos y peores;
"dolphin underwater" trae cientos. El control mira acentos, ñ y artículos
comunes en castellano. Es una heurística y lo pueden discutir.

## Las fotos

**Una foto por escena, sin repetir.** En los datos de prueba la escena 2 y la 4
comparten la mejor foto; la elección se queda con la segunda opción en la 4.
Por eso se pide `per_page=3` y no 1.

**Desfase.** Si llegan menos respuestas que escenas, **Elegir fotos** se
detiene con un error. Es el mismo control de longitud de la clase 6: emparejar
por posición sin verificar es la forma más silenciosa de romper un flujo.

**Lo que exige Unsplash** (y por qué el flujo tiene los nodos que tiene):

| Requisito | Dónde está |
| --- | --- |
| Usar la foto desde su dirección, sin copiarla | `src` del video apunta a `images.unsplash.com` |
| Conservar el parámetro `ixid` | se parte de `urls.raw`, que lo trae |
| Avisar cada uso | **Registrar uso de la foto**, a `download_location` |
| Nombrar al fotógrafo con enlace | escena de créditos en el video + `perfil` con `utm` |

El aviso de uso cuenta para el límite. Son **ocho pedidos por video**, no
cuatro: el sexto video en una hora ya choca con los 50.

## El render y el sondeo

Estados de JSON2Video: `pending` → `running` → `done`, o `error`. Un render de
30 segundos suele tardar entre uno y dos minutos.

| Estado | Rama |
| --- | --- |
| `done` | listo → costo |
| `error` | fallo → aviso |
| `pending`, `running` | en curso → ¿quedan vueltas? |

El lazo hace hasta **18 consultas de 20 segundos: seis minutos**. Después va al
aviso como `render_demorado`. Ese aviso dice que **no** hay que pedirlo de
nuevo: el render sigue del otro lado y probablemente se cobre igual.

**El error más frecuente de la clase:** en **Consultar estado**, leer el
identificador con `{{ $json.project }}`. Funciona en la primera vuelta y
falla en la segunda, porque lo que llega desde el lazo es la respuesta de la
consulta anterior, que tiene el identificador en `movie.project`. La forma
correcta es leerlo del nodo del pedido:
`{{ $('Pedir el render').first().json.project }}`.

**`$runIndex`** cuenta cuántas veces se ejecutó el nodo donde se lo usa, desde
cero. Por eso va en el IF del lazo y no en otro lado: si se pone en el Wait o en
la consulta, cuenta otra cosa.

## Costo de una pieza

Con el guion de prueba (69 palabras, unos 28 segundos de voz) más 3 segundos de
créditos, el estimado es **31** y el render real dura **31,4 s → 32 créditos**.

| | |
| --- | --- |
| Cuenta gratuita | 600 segundos, una vez |
| Por video | ≈ 32 |
| Videos por cuenta | **≈ 18** |
| Precio de referencia | US$ 16,63 / 3000 s ≈ US$ 0,0055 por segundo |
| Un video | ≈ US$ 0,18 |
| Uno por día, un mes | ≈ US$ 5,30 |

Verificar el precio actual en json2video.com/pricing antes de la clase.

El costo se calcula con `movie.duration` redondeado para arriba. La respuesta
también trae `consumed_credits`, que es un desglose por etapa, y
`remaining_quota.time`, que son los segundos que quedan en la cuenta: con eso
el flujo dice cuántas piezas más alcanzan.

**Los renders fallidos no se cobran.** Los demorados, sí, cuando terminan.

## La aprobación

El nodo **Pedir aprobación humana** es un Gmail con la operación **Send and
Wait for Response**: manda el correo con dos botones (Aprobar y Descartar) y
deja la ejecución en pausa hasta que llega el clic o se vence el plazo de un
día. El destinatario sale de `correo_revision`, en **El canal**.

**Marcar el tema** traduce la respuesta:

| Qué pasa | Lo que sale del Gmail | `decision` | Rama | ¿Se cierra el tema? |
| --- | --- | --- | --- | --- |
| Clic en **Aprobar** | `data.approved: true` | `aprobar` | aprobado | sí |
| Clic en **Descartar** | `data.approved: false` | `rechazar` | rechazado | sí |
| Nadie responde en un día | sin `data` | `vencido` | vencido | **no**, se reintenta |

**Probar la rama vencida en clase:** bajar el límite de espera del Gmail a
1 minuto, ejecutar y no tocar el correo. Una rama que nunca se ejecutó no está
probada.

**Hacer clic dos veces:** la segunda vez la ejecución ya terminó y n8n
responde que no hay nada esperando. No hay doble aprobación posible: es
idempotente por construcción, y vale la pena señalarlo con la clase 3 en mente.

**El correo llega desde la cuenta de cada alumno, a sí mismo.** Si alguien
deja tu correo en `correo_revision`, te llegan sus pedidos de aprobación y su
ejecución queda esperando un clic tuyo.

### Gmail con OAuth2, sin explicarlo todavía

La credencial de Gmail es OAuth2, y en una instancia propia eso pide un
cliente en Google Cloud. Hoy se hace como receta (diapositiva "Paso 0, segunda
parte"); la clase 8 explica qué se firmó y le agrega Drive al mismo cliente.

| Error | Causa |
| --- | --- |
| `redirect_uri_mismatch` | La dirección de retorno no es exactamente la que muestra n8n. Se copia, no se escribe. |
| `access_denied` / "app en prueba" | La cuenta no está agregada como usuario de prueba. |
| "Gmail API has not been used in project" | Falta habilitar la API en el proyecto. |
| "Google no verificó esta app" | Es esperable: es su propia app en prueba. Se continúa. |

Con la app en modo prueba, el permiso vence a los siete días. No pasa en la
clase; pasa con el TP, y es uno de los temas de la clase 8.

## Los datos estáticos

`Elegir tema`, `Registrar costo` y `Marcar el tema` usan
`$getWorkflowStaticData('global')`:

- `cerrados`: los temas aprobados o rechazados.
- `libro`: el libro de costos, las últimas 50 piezas.

**n8n solo guarda los datos estáticos en ejecuciones de producción** (flujo
activado, disparado por el Schedule). En las ejecuciones manuales se leen pero
no se guardan, y por eso a mano sale siempre `t-02`. Alguien lo va a reportar
como un error del flujo: no lo es, y es un buen momento para hablar de la
diferencia entre probar y operar, que es el tema de la clase que viene.

## Las fallas inducidas

**1. Una voz que no existe.** Cambiar `es-AR-ElenaNeural` por
`es-AR-ElenitaNeural` en **Armar el video**. Hay dos desenlaces posibles y el
flujo tiene una rama para cada uno: si JSON2Video lo detecta al recibir el
pedido, vuelve `success: false` y va al aviso como `pedido_rechazado`; si lo
detecta al renderizar, el pedido se acepta, devuelve un `project`, y el error
llega en el sondeo como `render_fallido`. **En ninguno de los dos se cobra.**
El segundo caso es la demostración de que en un proceso asincrónico el error
puede llegar minutos después de un pedido que "funcionó". Si sale el primero,
probar con un campo inventado dentro de un elemento (`"zoomm": 3`).

**2. Paciencia corta.** Bajar el lazo a `$runIndex < 2` y el Wait a 5
segundos. El render no llega a terminar y el flujo va a `render_demorado`.
Preguntar: ¿se cobró? Entrar a la cuenta de JSON2Video un par de minutos
después y ver que el video terminó igual. **Sí se cobró.**

**3. La clave de Unsplash sin `Client-ID `.** Un 401. Después, con la clave
bien, ejecutar varias veces seguidas hasta pasar los 50 pedidos: un 403 con
"Rate Limit Exceeded". El aviso distingue los dos casos por el texto, no por
el código; es la misma lección de los dos `403` de la clase 4.

## Sobre los créditos del aula

Con 600 segundos por alumno no hay margen para renderizar cada vez que se
prueba algo. La regla para dar en voz alta al principio del paso 2: **cuando
tengan un render bueno, fijen la salida de Consultar estado.** Lo que sigue
(costo, aprobación) se arma y se prueba contra ese resultado fijado.

Si alguien se queda sin créditos, puede seguir la práctica con la salida
fijada de un compañero: en el editor, pegar el JSON en el pin de **Consultar
estado**. El video del enlace va a ser el del compañero, y el resto del flujo
funciona igual.

## Lo que queda para la clase 8

La rama **Video aprobado** termina en un nodo vacío. La dirección del video
que devuelve JSON2Video no es permanente: la clase que viene lo bajan y lo
guardan en su propio Google Drive, con OAuth2. **Notificar la falla** también
es un nodo vacío, y la clase 8 lo convierte en un aviso que llega aunque nadie
esté mirando n8n.
