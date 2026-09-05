# Clase 2 — clave de casos

Uso del docente. **No repartir antes de la práctica.**

Tres fuentes, 22 registros en total. El resultado correcto es:

| Salida | Cantidad |
| --- | --- |
| Padrón unificado | **12** |
| Cuarentena | **4** |
| Descartados por consentimiento | **3** |
| Duplicados absorbidos | **3** |

`12 + 4 + 3 + 3 = 22`. Si la cuenta no cierra, algo se perdió por el camino.

## Defectos plantados, por registro

### Fuente A — formulario web

| Registro | Defecto | Qué tiene que pasar |
| --- | --- | --- |
| `web-0001` Ana Ferreyra | — | Entra. Duplicada con `C1` y `C5`. |
| `web-0002` Bruno Martinez | — | Entra. Duplicado con `B1`. |
| `web-0003` Carla Ojeda | `consent: false` | Descartado por consentimiento. |
| `web-0004` Diego Sosa | Espacios de sobra en nombre, apellido y correo; correo en mayúsculas mezcladas; teléfono en formato local con el 15 | Entra normalizado: `diego.sosa@ejemplo.com`, `+543415550121`. |
| `web-0005` Elena Ruiz | Correo sin dominio | Cuarentena, motivo `email_invalido`. |
| `web-0006` Facundo Ibarra | `phone` y `company` en `null` | Entra con esos campos en `null`. Un campo vacío no invalida el registro. |
| `web-0007` Gabriela Paz | Fecha con desplazamiento `-03:00` en vez de `Z` | Entra con `fecha_alta: 2026-03-03`. Ojo con quien convierta a UTC sin pensar: da el 3 igual, pero por poco. |
| `web-0008` Hugo Benitez | **No existe la clave** `contact.email` | Cuarentena, motivo `email_ausente`. Distinto de "vacío": la clave no está. |

### Fuente B — CRM

| Registro | Defecto | Qué tiene que pasar |
| --- | --- | --- |
| `4412` Bruno Martinez | Nombre en mayúsculas con espacio doble, correo con espacio al final | Se funde con `web-0002`. Gana la fecha del CRM (28/02, más antigua) y los datos de la web (mayor prioridad). |
| `4415` Hernan Quiroga | `opt_in: "no"` | Descartado por consentimiento. |
| `4418` Ivana Ledesma | Teléfono sin prefijo de país | Entra como `+543415550107`. |
| `4421` Jorge Da Silva | `opt_in: "YES"` en mayúsculas; `job_title` cadena vacía; apellido de dos palabras | Entra. `cargo: null`. El apellido correcto es "Da Silva": el que parta por el primer espacio lo va a romper. |
| `4423` Karina Nunez | `opt_in: null` | Cuarentena, motivo `consentimiento_ausente`. **Ausencia de consentimiento no es negativa: es indeterminada**, y por eso no va al mismo lado que Carla o Hernán. |
| `4427` Leandro Pinto | `created: "31/02/2026"`, fecha que no existe | Cuarentena, motivo `fecha_invalida`. La trampa: muchos parsers la convierten en 3 de marzo sin quejarse. |
| `4430` Maria Jose Alvez | Nombre compuesto de tres palabras | Entra. Ambiguo por diseño: "Maria Jose" + "Alvez" es lo razonable, pero la regla hay que declararla. |

### Fuente C — planilla del evento

| Registro | Defecto | Qué tiene que pasar |
| --- | --- | --- |
| `C1` Ana Ferreyra | Correo en mayúsculas; empresa abreviada distinto que en la web | Se funde con `web-0001`. Gana "Cooperativa El Molino" por prioridad de la web. |
| `C2` Nicolás Bravo | `"Si"` sin tilde | Entra. |
| `C3` Olga Ferrari | `"SI"` en mayúsculas; teléfono cadena vacía | Entra con `telefono: null`. |
| `C4` Pablo Rey | `"No"` | Descartado por consentimiento. |
| `C5` Ana Ferreyra | **Duplicado dentro de la misma fuente**, con espacio al final del correo | Se absorbe. La mayoría de los flujos deduplican entre fuentes y se olvidan de hacerlo dentro de una. |
| `C6` Rocío Vega | Espacio doble en el nombre; `Fecha` vacía | Entra con la fecha de ejecución, por la regla del esquema. |
| `C7` (sin nombre) | `Nombre y Apellido` vacío, correo válido | **Caso abierto a propósito.** El esquema dice que `nombre` es obligatorio, así que corresponde cuarentena; pero es discutible, porque la clave del padrón es el correo. Que el aula decida y deje la decisión escrita. En la cuenta de arriba está contado como **entra**, con nombre en `null`. |

## Padrón esperado, por correo

`ana.ferreyra`, `bruno.martinez`, `diego.sosa`, `facundo.ibarra`, `gabriela.paz`,
`ivana.ledesma`, `jorge.dasilva`, `maria.jose.alvez`, `nicolas.bravo`,
`olga.ferrari`, `rocio.vega`, `sergio.luna`.

## Cómo servir los datos

Los tres archivos se publican en la instancia de la cátedra como tres rutas
distintas. Si hace falta levantarlos a mano, alcanza con un flujo de n8n por
fuente: disparador de webhook, nodo de respuesta con el JSON pegado en el
cuerpo. También sirven como archivos sueltos leídos con un nodo de lectura,
pero conviene que sean tres llamadas HTTP para que el ejercicio se parezca al
caso real.

## Falla inducida, si sobra tiempo

Cambiar en la fuente B la clave `email_address` por `mail` y no avisar. El flujo
sigue corriendo sin error y produce siete registros sin correo. Es el mejor
ejemplo de por qué la validación va antes de la unión y no después.
