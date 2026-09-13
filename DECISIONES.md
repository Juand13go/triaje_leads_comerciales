# Decisiones de diseño — Radar

Registro de las decisiones importantes del proyecto, con su razón y sus consecuencias.
Se documentan aquí para que dentro de seis meses —o en una entrevista— se pueda explicar
**por qué** el sistema es como es, no solo cómo funciona.

---

## Estado del sistema

**Funciona:** recepción por Telegram, persistencia de conversaciones y mensajes, agente con
*tool calling* que extrae `respuesta_cliente`, `productos_interes`, `ciudad` y
`debe_escalar`, creación del lead, asignación al asesor menos cargado, notificación,
respuesta al cliente, panel web para listar asesores y sus leads, y registro del cierre en
`venta` / `no_venta`.

**Arquitectura:** tres capas (API, servicio, persistencia), PostgreSQL con SQLModel,
migraciones con Alembic, validación con Pydantic, orquestación con n8n, todo en Docker
Compose. Excepciones propias del dominio y registro de eventos con identificador de
conversación.

**Versión:** `v1.0.0`. El proyecto está terminado y congelado en este estado.

---

## D-01 · La escalación se decide por completitud del lead

**Contexto.** Después de que el modelo decide `debe_escalar`, el código lo sobrescribe:

```python
if not ciudad or not productos_interes:
    escalar = False
```

**Razón.** Evitar que los asesores reciban todo. Si se escala cada mensaje con molestia o
cada petición de hablar con alguien, la automatización no aporta nada. La compuerta
garantiza que solo suba lead calificado.

**Problema identificado.** La regla usa *"¿está completo el lead?"* para responder
*"¿necesita un humano?"*. Coinciden casi siempre, pero se separan en los casos caros: un
cliente de alta intención que aún no dio sus datos no se escala.

**Causa raíz.** La salida es binaria, así que solo se puede elegir entre dos reglas malas:
escalar de más e inundar a los asesores, o escalar de menos y perder clientes. Con una
salida continua, la completitud dejaría de ser una compuerta para ser una variable más que
baja la prioridad.

**Estado.** Se mantiene tal cual. Es el comportamiento conocido y documentado de esta
versión.

---

## D-02 · Un fallo del modelo escala a un humano

**Contexto.** El bloque `except` de `comunicacion_agente` devolvía `escalar: False` y un
mensaje pidiéndole al cliente que volviera más tarde.

**Problema.** Eso no filtra un lead malo: pierde un cliente porque la infraestructura falló,
y sin que nadie se entere. Es un problema distinto al de D-01 —modo degradado, no
calificación— y merece la respuesta contraria.

**Decisión.** Ante fallo del modelo, `escalar: True` y un mensaje al cliente avisando que un
asesor lo contactará. Como el esquema de entrada exige valores no vacíos, los campos que no
se alcanzaron a capturar viajan como `"Por confirmar"`, lo que además le indica al asesor
qué le falta preguntar.

> **Principio:** cuando la IA falla, el sistema no inventa ni abandona: entrega el caso a
> una persona.

---

## D-03 · El modelo es configuración, no código

**Contexto.** El sistema dejó de funcionar de un día para otro. El proveedor retiró
`llama-3.3-70b-versatile` y toda llamada devolvía `404 model_not_found`. El código no había
cambiado.

**Decisión.** El identificador del modelo sale a la variable de entorno `GROQ_MODEL`.
Cambiar de modelo no requiere tocar código, reconstruir la imagen ni hacer un commit.

**Modelo elegido:** `openai/gpt-oss-20b`. La tarea —extraer cuatro campos de un mensaje
corto— no requiere un modelo grande, y la latencia importa porque el cliente está esperando
en el chat.

**Consecuencia.** El proveedor del modelo es una dependencia externa que puede cambiar sin
aviso. Todo lo que determina el comportamiento del agente —modelo, *prompt*, definición de
las herramientas— debe poder cambiarse sin desplegar.

---

## D-04 · Sin ajuste fino: el problema es de conocimiento, no de estilo

**Decisión.** No se entrena ni se ajusta ningún modelo. El agente necesita conocer el
catálogo, no cambiar su forma de escribir, y para conocimiento lo correcto es darle acceso
a la información.

**Implementación.** El catálogo completo se inyecta en el *prompt* del sistema en cada
mensaje.

**Limitación aceptada.** Funciona con un catálogo pequeño. Con cientos de referencias, el
costo por mensaje crece y se topa con el límite de contexto. La solución sería búsqueda
semántica sobre el catálogo.

---

## D-05 · El agente responde solo sobre su dominio

**Decisión.** El agente no conversa sobre temas ajenos al negocio. Un asistente comercial
que responde de cualquier cosa es imposible de evaluar y de acotar.

**Implementación.** Ante una pregunta fuera de dominio no se responde con un rechazo seco:
se reconoce, se acota y se ofrece contacto humano. La restricción se diseña como parte del
producto, no como un `else`.

---

## D-06 · El sistema registra el resultado de cada decisión

**Decisión.** El panel del asesor cierra cada lead como `venta` o `no_venta`.

**Por qué importa.** Vincula cada decisión del sistema con un resultado de negocio real, y
hace que el propio sistema genere su conjunto de datos etiquetados en vez de depender de
datos externos. Sin ese registro, no hay forma de saber si las decisiones automáticas
estaban bien tomadas.

---

## D-07 · No todos los fallos son iguales

**Contexto.** Al hacer varias peticiones seguidas, el proveedor respondió con un límite de
tasa. La excepción cayó en el mismo `except` que atrapa cualquier otro error, y el sistema
reaccionó como si el fallo fuera permanente.

**Problema.** Un límite de tasa y un modelo retirado son cosas distintas:

| Tipo de fallo | Naturaleza | Respuesta correcta |
|---|---|---|
| Límite de tasa, tiempo de espera, error del servidor | Transitorio | Reintentar con espera creciente |
| Modelo retirado, clave inválida | Permanente | Escalar a un humano |

Atraparlos con el mismo `except` obliga a elegir una única respuesta que está mal para uno
de los dos casos.

**Decisión.** Los fallos transitorios se resuelven antes de llegar al `except`, mediante los
reintentos con espera exponencial del cliente HTTP (`max_retries=4`) y un `timeout` para no
dejar al cliente esperando indefinidamente. Así el `except` recupera su significado: cuando
se ejecuta, el fallo es real y corresponde escalar (D-02).

**Y un principio de producto:** un límite de tasa es una restricción entre el sistema y su
proveedor. El cliente no tiene por qué enterarse, y absorberla es responsabilidad del
sistema.

---

## Deuda técnica conocida

| Asunto | Detalle |
|---|---|
| Consultas N+1 | `menos_cargado` llama a `comparacion()` una vez por asesor. Con cuatro no importa; con cincuenta, sí. Se resuelve con una sola consulta agrupada. |
| Catálogo en cada mensaje | `catalogo_a_texto` consulta la base y arma el texto en cada llamada. Se puede cachear. |
| Sin autenticación | Los endpoints del panel están abiertos. Aceptable en local, no en un servidor. |
| Sin límite de tasa propio | `/procesar` consume cuota del proveedor en cada llamada. |
| `productos_interes` es texto libre | Sin normalizar contra el catálogo. Se resuelve con búsqueda semántica. |
| Imagen `n8n:latest` | Puede actualizarse sola y romper el entorno. Conviene fijar la versión. |
| *Prompt* dentro del código | Vive en `conversacion.py`, sin historial propio ni forma de evaluarlo. |
| Sin pruebas automatizadas | No hay conjunto de casos para verificar el comportamiento del agente. |