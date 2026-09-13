# TRIAJE LEADS COMERCIALES/RADAR
Este sistema plantea la automatización de la atención y la clasificación de las solicitudes comerciales que le entran a una empresa por sus canales de mensajería.
El flujo inicia con un mensaje del cliente por alguno de los canales de la empresa (actualmente Telegram), la información de la conversación es almacenada en una BD;
esta información se recibe y procesa en el archivo de Python (donde vive el agente de IA), este archivo entrega la respuesta del agente junto con los datos del lead
(productos de interés, ciudad y si debe escalarse o no), luego esta respuesta es almacenada en la BD. El flujo determina si escalar el lead a un humano (creando el lead,
asignando un asesor y alertándolo) o entregar la respuesta él mismo; finalmente se le responde al cliente por el mismo canal por el que envió el mensaje (humano o agente)
y se actualiza el estado de la conversación. Cuando el asesor termina de atender el lead lo cierra desde el panel como venta o no venta, y ese resultado también queda
guardado, de manera que cada decisión que tomó el sistema queda ligada a lo que pasó realmente con ese cliente.

## Arquitectura
PostgreSQL BD (Sistema gestor de datos, almacena conversaciones, mensajes, leads, asesores y el catálogo de productos)
n8n (Tecnología para la automatización de flujos y tareas repetitivas (BASE DEL PROYECTO))
FastAPI - Python (Framework - Usado para la lógica del proyecto)
Groq API (LLM, agente IA - con tool calling para estructurar la respuesta y la decisión de escalación; el modelo se define en la variable de entorno GROQ_MODEL y no está escrito dentro del código)
Docker (Herramienta esencial para mantenibilidad, una arquitectura limpia y para fácil acceso al proyecto en cualquier máquina)
Migraciones con Alembic
Schemas de Pydantic (Validación de los datos que entran y salen)
SQLModel (ORM - Definición de las tablas y comunicación con PostgreSQL)
Cloudfared (Proxy inverso para exponer n8n por HTTPS y recibir el webhook de Telegram)
Frontend (HTML, CSS y JS) para monitoreo y cierre de los leads

## Requisitos
El sistema corre en Docker, esta tecnología se encarga de que el sistema funcione sin tener que instalar nada; las dependencias del proyecto están en el
requirements.txt.
Necesitas: Docker y Docker Compose, un bot de Telegram (creado con @BotFather) y una API Key de Groq.

## Configuración
En la raíz del proyecto está el archivo .env.example con todas las variables que necesita el sistema, sin valores. Se copia como .env y se completa:
POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD (PostgreSQL)
N8N_DB, N8N_USER, N8N_PASSWORD (Base de datos y credenciales de n8n)
WEBHOOK_URL (URL HTTPS que entrega cloudfare)
GROQ_API_KEY (Agente IA)
GROQ_MODEL (Modelo que usa el agente; está por fuera del código para poder cambiarlo sin tocar nada ni reconstruir la imagen)
TELEGRAM_BOT_TOKEN (Token del bot)
COMPOSE_PROJECT_NAME (Nombre del proyecto en Docker; fija los nombres de los volúmenes para que renombrar o mover la carpeta no rompa la persistencia de la BD)

El .env nunca se sube al repositorio, por eso existe el .env.example: documenta qué hace falta sin exponer los valores.
El token del bot de Telegram y el consumer key/secret de WooCommerce (sincronización del catálogo) se configuran como credenciales dentro de n8n.

## Cómo levantar el entorno
Clonar el repositorio
Crear el .env a partir del .env.example (ver sección de configuración)
Para el proxy inverso con cloudfare: "cloudflared tunnel --url http://localhost:5678" (Recibirás una URL HTTPS como esta: "https://vitamin-barrier-odds-performing.trycloudflare.com", colócala en la variable WEBHOOK_URL del .env)
docker compose up (al arrancar se ejecutan automáticamente las migraciones con Alembic y el seed.py que puebla el catálogo de productos y los asesores)
Abrir n8n en la URL de cloudfare (HTTPS) e importar el archivo con el flujo (carpeta n8n/)
Configurar la credencial de Telegram en n8n y activar el flujo
La documentación interactiva de la API queda disponible en http://localhost:8000/docs y el panel de los asesores en http://localhost:8000/static/index.html

## Estructura del proyecto
El Dockerfile construye el servicio de FastAPI.
La estructura de este proyecto está guiada por 3 capas (Servicio, Persistencia y API), y las tres viven dentro de fastapi/app/. En la capa de servicio encuentras toda la
lógica de la aplicación (en Python nativo), incluyendo la comunicación con el agente de IA (conversacion.py); en la capa de persistencia encuentras todos los queries y la
comunicación de la aplicación con la base de datos (repositorio.py); y finalmente tenemos la capa de API con todos los endpoints de FastAPI (rutas.py) y los schemas de
Pydantic (schemas.py). En esa misma carpeta app/ está excepciones.py, donde se definen las excepciones propias del dominio.
Los endpoints expuestos son: /conversacion (busca o crea la conversación), /historial (mensajes de una conversación), /guardar_mensaje, /procesar (ejecuta el agente IA),
/crear_lead (crea el lead y le asigna el asesor menos cargado), /estado, /listar_asesores, /leads_por_asesor (los leads asignados a un asesor) y /cerrar_lead (registra el
cierre como venta o no venta).
En la raíz de fastapi/ están models.py (definición de las tablas con SQLModel), database.py (conexión a la BD), seed.py (inyección del catálogo desde productos.json y de
los asesores desde asesores.json), main.py (punto de entrada de la aplicación), la carpeta alembic/ con las migraciones y la carpeta static/ con el frontend del panel
(index.html, script.js y style.css).
docker-compose.yml: Configuración del Docker y comandos de arranque y montaje de la BD (creación del esquema, inyección de datos a la BD (seed.py), arranque de la aplicación).
Los volúmenes y la red están declarados con nombre explícito para que no dependan del nombre de la carpeta.
init.sql: Crea la base de datos exclusiva de n8n al levantar PostgreSQL por primera vez.

## Estado del proyecto
Este repositorio está terminado y congelado en la versión v1.0.0. Todo lo que hay acá fue desarrollado por mí.
El trabajo continúa en https://github.com/Juand13go/radar_motor_difuso, donde la decisión de escalación (que hoy es binaria y la toma el modelo) se reemplaza por un motor
de lógica difusa que asigna una prioridad continua y puede explicar con qué reglas llegó a ella.
Las decisiones de diseño que se tomaron durante el desarrollo, con la razón de cada una, están en DECISIONES.md.