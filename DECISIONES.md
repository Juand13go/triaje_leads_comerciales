# Decisiones de diseño
Acá quedan escritas las decisiones importantes que tomé durante el desarrollo y la razón de cada una. La idea es poder explicar más adelante por qué el sistema quedó
así y no de otra forma, y no tener que reconstruirlo de memoria.

## La escalación se decide por la completitud del lead
Después de que el modelo decide si el lead debe escalarse, el código sobrescribe esa decisión: si falta la ciudad o faltan los productos de interés, no se escala.
La razón es evitar que los asesores reciban todo. Si se escalara cada mensaje donde el cliente se muestra molesto o pide hablar con alguien, la automatización no
aportaría nada y el asesor terminaría atendiendo lo mismo que antes; la compuerta garantiza que solo suba lead calificado.
El problema de esa regla es que está usando una pregunta para responder otra: mide si el lead está completo, pero lo que quiere decidir es si el caso necesita un humano.
Casi siempre coinciden, pero se separan justo en los casos caros, como un cliente con clara intención de compra que todavía no ha soltado sus datos.
La causa de fondo es que la salida es binaria, y siendo binaria solo se puede elegir entre dos reglas malas: escalar de más e inundar a los asesores, o escalar de menos
y perder clientes. Con una salida continua la completitud dejaría de ser una compuerta y pasaría a ser una variable más que baja la prioridad, sin bloquear nada.
Se deja tal cual en esta versión, porque es el comportamiento conocido y documentado del sistema.

## Cuando el modelo falla, el lead se escala a un humano
El bloque except de comunicacion_agente devolvía escalar en False y un mensaje pidiéndole al cliente que volviera a escribir más tarde.
Eso no estaba filtrando un lead malo: estaba perdiendo un cliente porque la infraestructura falló, y sin que nadie se enterara. Es un problema distinto al de la decisión
anterior (ahí se trata de calificar el lead, acá de que el sistema no está funcionando) y por eso merece la respuesta contraria.
Ahora, ante un fallo del modelo, se escala. Como el schema de entrada de /crear_lead exige que los campos no vengan vacíos, los datos que no se alcanzaron a capturar
viajan como "Por confirmar", lo que de paso le indica al asesor qué es lo que le falta preguntarle al cliente.
La regla que queda es que cuando la IA falla el sistema no inventa ni abandona, sino que entrega el caso a una persona.

## El modelo es configuración, no código
El sistema dejó de funcionar de un día para otro sin que yo hubiera tocado nada: Groq retiró el modelo llama-3.3-70b-versatile y todas las llamadas empezaron a devolver
404 model_not_found.
Por eso el identificador del modelo salió del código a la variable de entorno GROQ_MODEL. Cambiar de modelo ahora no implica editar un archivo, reconstruir la imagen ni
hacer un commit.
El modelo que quedó es openai/gpt-oss-20b. La tarea que hace el agente (leer un mensaje corto y devolver cuatro campos estructurados) no necesita un modelo grande, y la
latencia sí importa porque el cliente está esperando la respuesta en el chat.
La lección que dejó el incidente es que el proveedor del modelo es una dependencia externa que puede cambiar sin avisar, y que todo lo que determina el comportamiento del
agente (el modelo, el prompt, la definición de las herramientas) debería poder cambiarse sin desplegar.

## No todos los fallos son iguales
Haciendo varias peticiones seguidas, el proveedor respondió con un límite de tasa. Esa excepción caía en el mismo except que atrapa cualquier otro error, así que el
sistema reaccionó como si el fallo fuera permanente.
Pero un límite de tasa y un modelo retirado no son lo mismo. El primero es transitorio y lo correcto es reintentar; el segundo es permanente y lo correcto es escalar.
Atrapar los dos con el mismo except obliga a elegir una sola respuesta que necesariamente está mal para uno de los dos casos.
La solución fue que los fallos transitorios se resuelvan antes de llegar al except, usando los reintentos con espera exponencial del propio cliente (max_retries=4) y un
timeout para no dejar al cliente esperando indefinidamente. Así el except recupera su significado: si se ejecuta, el fallo es real y corresponde escalar.
Y hay algo de producto ahí también: un límite de peticiones por minuto es un acuerdo entre el sistema y su proveedor. El cliente que está escribiendo por Telegram no
tiene por qué enterarse de eso, y absorberlo es responsabilidad del sistema.

## Sin fine-tuning, porque el problema es de conocimiento y no de estilo
No se entrena ni se ajusta ningún modelo. Lo que el agente necesita es conocer el catálogo, no cambiar su forma de escribir, y para eso lo correcto es darle acceso a la
información, no reentrenarlo. Hacer fine-tuning habría sido más caro y habría resuelto el problema equivocado.
Hoy el catálogo completo se arma como texto y se inyecta dentro del prompt del sistema en cada mensaje. Esto funciona bien con un catálogo pequeño, pero con cientos de
referencias el costo por mensaje crece y se llega al límite de contexto; la salida a eso sería una búsqueda semántica sobre el catálogo.

## El agente responde solamente sobre su dominio
El agente no conversa de temas ajenos al negocio. Un asistente comercial que responde de cualquier cosa es imposible de acotar y de evaluar, y además abre la puerta a
que diga cosas que la empresa no quiere decir.
La restricción está diseñada como parte del producto y no como un rechazo: ante una pregunta que se sale del dominio, el agente lo reconoce, lo acota y ofrece el
contacto con un asesor.

## El sistema registra el resultado de cada decisión
Desde el panel, el asesor cierra cada lead como venta o no venta.
Esto es lo que amarra cada decisión que tomó el sistema con lo que pasó de verdad con ese cliente. Sin ese registro no hay manera de saber si las decisiones automáticas
estaban bien tomadas, y el sistema termina generando su propio conjunto de datos en vez de depender de datos externos.

## Deuda técnica conocida
menos_cargado llama a comparacion() una vez por cada asesor, o sea una consulta por asesor. Con cuatro no se nota, con cincuenta sí; se resuelve con una sola consulta
agrupada.
catalogo_a_texto consulta la base y arma el texto del catálogo en cada mensaje, cuando se podría cachear.
Los endpoints del panel no tienen autenticación. Es aceptable corriendo en local, no lo sería en un servidor.
/procesar consume cuota del proveedor en cada llamada y no tiene un límite de tasa propio.
productos_interes se guarda como texto libre, sin normalizar contra el catálogo; es lo que resolvería la búsqueda semántica.
La imagen de n8n está en latest, así que se puede actualizar sola y romper el entorno; convendría fijar la versión.
El prompt del agente vive dentro de conversacion.py, sin historial propio ni manera de evaluar si un cambio lo mejoró o lo empeoró.
No hay pruebas automatizadas ni un conjunto de casos para verificar el comportamiento del agente.