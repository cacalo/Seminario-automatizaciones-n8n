# Clase 8 — clave de casos

Uso del docente. **No repartir antes de la práctica.**

La clase parte del flujo de la clase 7. El que lo terminó sigue con el suyo; el
que no, importa **8 - Fabrica en operacion - Template**, que es el resuelto de
la clase 7 (con la aprobación por Gmail) más el disparador de todas las
mañanas.

Archivos:

| Archivo | Qué es |
| --- | --- |
| `8 - Fabrica en operacion - Template.json` | Punto de partida: resuelto de la clase 7 + Schedule de las 9 |
| `Clase 8 - Fabrica en operacion.json` | La fábrica con entrega a Drive y fallas visibles |
| `Clase 8 - Avisar fallas.json` | El flujo de errores |
| `autoalojado/` | La demostración: `docker-compose.yml`, `Caddyfile`, `env.ejemplo` |

## Antes de la clase

- El cliente OAuth de Google Cloud **ya existe**: cada alumno lo creó en la
  clase 7 para Gmail. Hoy se le agrega Drive. Si alguien no llegó a hacerlo,
  tiene que hacer el Paso 0 completo de la clase 7 antes de empezar.
- Cada alumno usa **su propio proyecto y su propio cliente**. Un cliente
  compartido funciona, pero entonces el docente es dueño de la app a la que
  todos le dieron permiso sobre su Drive y su correo: es exactamente lo que la
  clase dice que no hay que hacer.

## Probablemente arranque con una falla real

Si entre la clase 7 y la 8 pasó una semana, **la credencial de Gmail de la
clase 7 venció**: con la app en modo prueba, Google invalida el permiso a los
siete días. El correo de aprobación no sale y el Gmail devuelve un error de
autenticación.

No hace falta inducir nada: es la mejor apertura posible para la sección
Delegar. La lección se cuenta sola. Reconectar la credencial (*Sign in with
Google* de nuevo) y seguir.

## OAuth2: lo que tiene que pasar hoy

1. En el **mismo proyecto** de Google Cloud de la clase 7, habilitar
   **Google Drive API**.
2. En n8n, credencial **Google Drive OAuth2 API** con el mismo identificador y
   secreto que la de Gmail.
3. *Sign in with Google*, leyendo esta vez la pantalla de permisos.

No hace falta tocar la pantalla de consentimiento ni el cliente: la dirección
de retorno es la misma para todas las credenciales de la instancia.

| Error | Causa |
| --- | --- |
| "Google Drive API has not been used in project" | Falta habilitar Drive en el proyecto. |
| `access_denied` | Se usó otra cuenta de Google, que no está como usuario de prueba. |
| La ventana se cierra y la credencial no conecta | Bloqueador de ventanas emergentes del navegador. |

**Lo que pide la credencial de Drive.** La credencial de n8n pide acceso amplio
al Drive y no se puede achicar desde ahí. Es el momento de la clase para
volver al token de la clase 4 (un solo permiso, gists) y comparar. La
alternativa razonable para un uso real es una cuenta de Google aparte para la
automatización.

**Los siete días.** Con la app en modo prueba, Google invalida el token de
renovación a los siete días. Es la falla que más se va a ver durante el TP: el
flujo anda una semana y después Drive y Gmail devuelven `401`. Se arregla
reconectando la credencial. Hoy probablemente ya lo vieron con Gmail; que el
flujo de errores sea lo que les avise la próxima vez.

Hay un detalle que conviene anticipar: **si vence la credencial de Gmail, el
aviso de la falla también falla**, porque el flujo de errores usa la misma.
Es el caso de la nota adhesiva "Si el aviso falla". Para el TP alcanza con
saberlo; en un sistema real el aviso iría por un canal que no dependa de la
misma credencial.

## La entrega

```
Que decidio (aprobado)
  → Bajar el video        HTTP, respuesta en modo File
  → Guardar en Drive      upload, nombre "AAAA-MM-DD - Título.mp4"
  → Guardar la ficha      create from text, "AAAA-MM-DD - Título.txt"
  → Pieza entregada       los dos enlaces de Drive
```

La ficha con los datos de prueba queda así:

```
Cómo duermen los delfines

Los delfines duermen con medio cerebro por vez.

#ciencia #delfines

Fotos de Lucía Paz, Ana Ito, Sara Kim, Juan Gil en Unsplash
Voz y edición: JSON2Video.
```

**Sin el modo File**, el HTTP intenta leer el video como texto y el nodo de
Drive falla diciendo que no encuentra el binario `data`. Es el error más
frecuente del paso 3 y se ve enseguida.

**La carpeta.** Los nodos vienen apuntando a la raíz del Drive. Que cada uno
cree una carpeta "Fábrica" y la elija en los dos nodos: si la eligen en uno
solo, el video y la ficha quedan separados.

**Probar sin gastar créditos.** Con la salida de **Consultar estado** fijada
desde la clase 7 (o la de **Pedir aprobación humana**, que en el template y el resuelto de
esta clase viene fijada con `{ "data": { "approved": true } }`, lo mismo que
devuelve el Gmail al hacer clic en Aprobar), la entrega se prueba completa contra el video ya
renderizado. Ojo: los datos fijados solo valen en ejecuciones manuales. En
producción se ignoran y todo corre de verdad, render incluido. Si la dirección del video ya venció (la de JSON2Video no es
permanente), el HTTP devuelve un error: es la demostración en vivo de por qué
existe este paso.

## Atrapado no es avisado

Es el concepto central de la clase y conviene demostrarlo en este orden:

1. Con el flujo de la clase 7 tal cual (rama de error terminando en un nodo
   vacío), conectar el flujo de errores en Settings, romper la clave de
   Unsplash y disparar en producción.
2. **No llega ningún correo.** En la lista de ejecuciones, la ejecución figura
   como exitosa: la rama de error la atrapó y terminó bien.
3. Reemplazar **Notificar la falla** por **Detener con error** (Stop and
   Error), repetir.
4. Llega el correo, con el mensaje que armó el flujo:

```
error_de_nodo: 401 - "OAuth error: The access token is invalid"
→ Credencial: revisar la clave de ese servicio.
```

El flujo de errores distingue dos casos por el último nodo ejecutado:

| `lastNodeExecuted` | Asunto | Significado |
| --- | --- | --- |
| `Detener con error` | `[Fábrica] ...` | Una falla prevista: el flujo la atrapó, la explicó y la marcó. |
| cualquier otro | `[Fábrica · imprevisto] ...` | Algo que ninguna rama de error cubría. Es lo más importante de leer. |

## La falla en producción (Paso 3)

**Por qué Unsplash.** Falla antes del render. Romper JSON2Video también
funcionaría, pero si alguien rompe otra cosa (el JSON del video, por ejemplo),
el render puede fallar después de aceptado o, peor, salir y cobrarse. La regla
para dar en voz alta: **se rompe lo que está antes de lo caro.**

**El Schedule.** Usa los campos de hora y minuto del nodo. Poner la hora
actual más dos minutos, activar, cerrar el editor. La zona horaria es la de la
instancia (`GENERIC_TIMEZONE`): si alguien lo pone a una hora que no dispara,
probablemente la instancia esté en UTC.

**Gemini sí se llama** antes de que falle Unsplash. Es gratis, pero cuenta para
la cuota diaria. Si el aula repite la prueba muchas veces, puede aparecer un
`429` de Gemini: es otra falla real, y el correo la va a traer como
`error_de_nodo`.

**Al terminar:** arreglar la clave y volver el Schedule a las 9. Si alguien
deja el Schedule cada pocos minutos y la clave arreglada, la fábrica renderiza
un video cada pocos minutos hasta quedarse sin créditos.

## La demostración de autoalojamiento

Los alumnos no la hacen: siguen trabajando en la instancia del seminario. Lo que
tienen que llevarse:

| Pieza | Para qué |
| --- | --- |
| Caddy | HTTPS automático. Sin HTTPS no hay OAuth2 ni webhooks útiles. |
| PostgreSQL con volumen | Flujos, credenciales, ejecuciones en espera. Un Wait de 24 h vive acá. |
| `N8N_ENCRYPTION_KEY` | Cifra las credenciales. Perderla inutiliza los respaldos. |
| `WEBHOOK_URL` | De acá salen `$execution.resumeUrl` y la dirección de retorno de OAuth2. |
| `GENERIC_TIMEZONE` | La hora a la que corre el Schedule. |
| `EXECUTIONS_DATA_PRUNE` | Sin esto, la base crece con cada ejecución. |
| Versión fijada | `latest` actualiza en cualquier reinicio. |

**Una conexión que vale la pena hacer en voz alta:** cada pieza de esa tabla
apareció en alguna clase como una cosa que "simplemente funcionaba". El
webhook de la clase 3, la reanudación de la clase 7 y el OAuth2 de hoy
funcionan porque alguien configuró `WEBHOOK_URL` y un certificado.

`docker compose config` sobre la carpeta valida el archivo sin levantar nada,
si se quiere mostrar sin un servidor a mano.

## Defensa del TP II

Las dos diapositivas de la defensa están escritas a partir de los temas del
seminario. **Ajustarlas a la consigna del TP II antes de la clase.** La de las
seis preguntas funciona bien proyectada mientras preparan: es el seminario
resumido en lo que un evaluador va a preguntar.
