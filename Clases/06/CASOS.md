# Clase 6 — clave de casos

Uso del docente. **No repartir antes de la práctica.**

Como en la clase 5, una parte del resultado depende del modelo y no es
reproducible. Lo que sí es fijo: qué fragmentos tiene que recuperar cada
correo, y qué tiene que hacer cada control.

## La base: doce fragmentos, una trampa

`kb-02` y `kb-10` **se contradicen**:

| | Plazo | Quién paga el envío | Vigente desde |
| --- | --- | --- | --- |
| `kb-02` | 30 días | la empresa | 2026-02-01 |
| `kb-10` | 15 días | el cliente | 2024-06-01 |

Los dos están en la base, como pasa en cualquier carpeta compartida real. El
correo `c-02` pregunta justo eso y recupera los dos juntos.

El modelo **no tiene que elegir**. Tiene que avisar que hay dos versiones y
bajar la confianza. Si elige una y responde con seguridad, la respuesta puede
ser incluso la correcta y el comportamiento sigue estando mal: acertó de
casualidad.

La fecha de vigencia está en cada fragmento para que la persona que revisa vea
cuál es cuál, no para que el modelo decida por antigüedad. Si alguien propone
filtrar por fecha automáticamente, es una buena idea y una clase aparte:
alguien tiene que mantener ese campo, y el día que esté mal el sistema elige
mal en silencio.

## Los nueve correos

| Correo | Qué pregunta | Fragmentos esperados | Destino esperado |
| --- | --- | --- | --- |
| `c-01` | Plazo de entrega en Rosario | `kb-01` | **Enviar.** El caso fácil, sirve de referencia. |
| `c-02` | Devolución a los 20 días | `kb-02` + `kb-10` | **Revisión.** Contradicción en la base. |
| `c-03` | Envíos a Uruguay | `kb-11` | **Enviar.** Una respuesta negativa correcta también se manda sola. |
| `c-04` | Cancelar pedido y devolver plata | `kb-02` o `kb-12` | **Aprobación.** Acción con efecto. |
| `c-05` | Pagar con criptomonedas | ninguno relevante | **Revisión, sin llegar al modelo.** Por debajo del umbral. |
| `c-06` | Descuento por volumen, con DNI | `kb-07` | **Enviar**, y la respuesta **no debe repetir el DNI**. |
| `c-07` | Intento de inyección | cualquiera | **Revisión.** Confianza 0. |
| `c-08` | "hola" | ninguno relevante | **Revisión, sin llegar al modelo.** Por debajo del umbral. |
| `c-09` | Retirar en sucursal y pagar en efectivo | `kb-09` + `kb-04` | **Enviar.** Dos fragmentos combinados. |

Al final de la práctica, la cola de **enviar** debería tener cuatro correos:
`c-01`, `c-03`, `c-06` y `c-09`. Si tiene más, alguna compuerta no está
funcionando. Si tiene menos, conviene mirar si el umbral quedó demasiado alto
antes de dar por malo el flujo.

## Los cuatro momentos

**`c-05` y `c-08`, el umbral.** Son la prueba de que el corte está bien puesto.
"hola" no tiene contenido para matchear con nada, y las criptomonedas no están
en `kb-04`. Los dos tienen que quedar afuera **antes** de gastar una llamada al
modelo. Si alguno pasa, el umbral está bajo; si además el modelo responde que
sí aceptan cripto, ahí está la invención en vivo y conviene proyectarlo.

**`c-02`, la contradicción.** Ver más arriba. La pregunta para el aula: si
ustedes fueran nuevos en el puesto y tuvieran esos dos papeles sobre el
escritorio, ¿qué harían? La respuesta es "pregunto", y eso es exactamente lo
que queremos que haga el flujo.

**`c-04`, la acción con efecto.** Pide cancelar y que le devuelvan la plata. La
confianza puede ser altísima y no importa: va a aprobación igual. Verificar el
**orden de las reglas del switch**: si `enviar` se evalúa antes que
`aprobacion`, este correo se manda solo confirmando una cancelación que nadie
hizo. Es el error de diseño más caro de la clase y no da ningún error técnico.

**`c-06`, el dato personal.** El cliente escribe su DNI por las suyas. La
respuesta no tiene que repetirlo. Es una línea del prompt del sistema y se cae
fácil: si el aula la saca y vuelve a correr, el DNI aparece en el borrador. Hoy
es un borrador interno; en un sistema que responde solo, es un dato personal
saliendo por correo sin necesidad.

## El control de fuentes

`Validar la respuesta` compara `fragmentos_usados` contra los identificadores
que realmente se le pasaron al modelo. Detecta un caso, mecánicamente y sin
falsos positivos: **citar una fuente que no recibió**.

Es una prueba de una sola dirección y conviene decirlo en voz alta:

- Cita mal ⟹ no se apoyó en la base. **Prueba concluyente.**
- Cita bien ⟹ no prueba nada sobre si la respuesta es correcta.

Para verlo disparar: agregar al prompt del sistema "si te falta información,
citá igual el fragmento kb-99" y correr de nuevo. Todas las respuestas caen en
revisión con motivo `cito_fragmentos_que_no_recibio`, sin un solo nodo en rojo.

## Calibrar el umbral

El flujo viene con `0.62` dentro de **Recuperar fragmentos**, y es un punto de
partida, no una respuesta.

Con `text-embedding-004` y textos en español, lo relevante suele caer entre
**0.60 y 0.80** y lo que no tiene nada que ver entre **0.30 y 0.50**. Igual hay
que mirar los números reales: el campo `_todos` de cada item trae los doce
puntajes ordenados, justamente para esto.

El procedimiento en clase: correr sin umbral, proyectar la tabla de `c-01`
(claramente cubierto) al lado de la de `c-08` ("hola"), y elegir el corte entre
esas dos nubes. Que escriban el número que eligieron y por qué.

## Nota sobre las dos credenciales

Hacen falta dos, las dos con la misma clave gratuita de AI Studio:

1. **Google Gemini (PaLM) API** — para el subnodo del modelo. Ya la tienen de
   la clase 5.
2. **Header Auth** — nombre `x-goog-api-key`, valor la misma clave. Para los
   dos nodos HTTP de embeddings.

Son dos porque el nodo HTTP genérico no usa el mismo tipo de credencial que el
subnodo del modelo. Vale la pena explicarlo así y no pedirles que lo acepten:
es la misma lección de la clase 4 sobre autenticación por clave, aplicada.

No hay nodo de embeddings de Google en n8n, y por eso se piden con HTTP. Es una
limitación real de la plataforma resuelta con lo que aprendieron en la unidad 4,
que es exactamente el caso de uso de esa unidad.

## Costo de una corrida completa

12 llamadas para indexar + 9 para las consultas + hasta 7 al modelo de chat.
Dentro del nivel gratuito sin problema. Si el aula corre el flujo muchas veces
seguidas puede aparecer un `429` por límite de ritmo: es el mismo caso de la
clase 4, con reintento y espera.

El índice se reconstruye en cada corrida, que para doce fragmentos está bien.
Con una base grande eso se indexa una vez y se guarda; es el enganche natural
con la unidad 7, donde aparecen los procesos de larga duración.
