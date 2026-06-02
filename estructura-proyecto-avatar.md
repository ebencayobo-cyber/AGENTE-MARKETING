# Estructura oficial del proyecto Avatar Influencer IA
> Documento de referencia. Toda decisión de diseño está aquí.
> Última actualización: 2026-05-29

---

## Stack tecnológico

| Componente | Herramienta |
|---|---|
| Orquestador | n8n Cloud 2.21.6 |
| Canal de administración | Telegram Bot |
| LLM texto | Groq API — modelo `llama-3.3-70b-versatile` |
| Generación de imágenes avatares | Pixazo API — modelo `flux-1-schnell` |
| Generación de banners publicitarios | OpenAI API — modelo `gpt-image-1` |
| Base de datos | Google Sheets |
| Almacenamiento de imágenes | Google Drive |
| Variables de entorno | n8n Settings → Variables |

---

## Variables de entorno en n8n

| Variable | Descripción |
|---|---|
| `GROQ_API_KEY` | API Key de Groq |
| `GOOGLE_SHEET_ID` | ID del Google Sheet principal |
| `PIXAZO_API_KEY` | API Key de Pixazo (generación de imágenes de avatar) |
| `OPENAI_API_KEY` | API Key de OpenAI (generación de banners publicitarios) |
| `USER_ID_TELEGRAM` | ID de Telegram del admin |

---

## Comandos de Telegram (admin)

| Comando | Sintaxis | Descripción |
|---|---|---|
| `/avatar` | `/avatar [id]` | Cambia el avatar activo en la sesión. Se guarda en la hoja `config`. |
| `/instruccion` | `/instruccion [texto]` | Enseñarle algo al avatar activo. Se guarda en Sheets automáticamente. El tema se clasifica con IA. |
| `/instruccion_off` | `/instruccion_off [id]` | Desactiva una instrucción por su ID sin borrarla. |
| `/post` | `/post [red_social] [tema]` | Genera un borrador de post + imagen. Lee perfil + instrucciones activas + historial. |
| `/historial` | `/historial [n]` | Muestra los últimos N posts generados. |
| `/publicar` | `/publicar [brand_name]` | Genera un banner publicitario para la marca indicada. Lee los datos de la hoja `publicidad`, descarga las imágenes de Drive y llama a OpenAI gpt-image-1 con las referencias visuales. |
| `/ayuda` | `/ayuda` | Muestra el menú de comandos. |

---

## Google Sheets — estructura de tablas

### Reglas generales
- Ninguna columna se rellena manualmente por el admin
- Todo se escribe automáticamente desde n8n
- El admin solo puede editar la hoja `perfil_avatar` si necesita ajustar datos base del personaje
- El admin edita manualmente la hoja `publicidad` para cargar los datos de cada marca/campaña
- La hoja `config` la gestiona n8n al recibir el comando `/avatar`

---

### Tabla: `perfil_avatar`

Una fila por avatar. Se crea manualmente una sola vez al crear el avatar.

| Columna | Tipo | Descripción | Ejemplo |
|---|---|---|---|
| `avatar_id` | string | Identificador único del avatar | `sofia_01` |
| `nombre` | string | Nombre del personaje | `Sofía Reyes` |
| `edad` | number | Edad ficticia | `24` |
| `ciudad` | string | Ciudad ficticia del personaje | `Madrid` |
| `bio` | string | Descripción del perfil público | `Diseñadora freelance amante del café y los viajes` |
| `tono_voz` | string | Cómo habla el avatar | `casual, cercana, con humor sutil` |
| `intereses` | string | Temas que toca en sus publicaciones | `moda, viajes, café, diseño` |
| `frases_tipicas` | string | Muletillas características del personaje | `"Total", "La verdad es que..."` |
| `lo_que_nunca_dice` | string | Restricciones de contenido | `no usa groserías, no habla de política` |
| `redes_activas` | string | Redes donde publica | `instagram, tiktok` |
| `carpeta_drive` | string | URL de la carpeta de imágenes en Drive | `https://drive.google.com/...` |
| `img_prompt_base` | string | Descripción física del avatar para prompts de imagen | `24 year old spanish woman, brown hair, green eyes, casual modern style` |
| `img_prompt_estilo` | string | Estilo visual consistente para imágenes | `realistic photo, natural lighting, high quality, instagram influencer` |
| `img_prompt_negativo` | string | Lo que nunca debe aparecer en las imágenes | `cartoon, anime, ugly, deformed` |

---

### Tabla: `config`

Una fila por clave de configuración. Gestiona el estado global del sistema.

| Columna | Tipo | Quién la llena | Descripción | Ejemplo |
|---|---|---|---|---|
| `clave` | string | n8n automático | Identificador de la configuración | `avatar_activo` |
| `valor` | string | n8n automático | Valor actual | `sofia_01` |
| `actualizado` | string | n8n automático | Fecha y hora del último cambio | `2026-05-22 14:30` |

**Filas iniciales requeridas:**

| clave | valor |
|---|---|
| `avatar_activo` | `sofia_01` |

---

### Tabla: `instrucciones`

Crece con cada `/instruccion` que el admin envía por Telegram.

| Columna | Tipo | Quién la llena | Descripción | Ejemplo |
|---|---|---|---|---|
| `id` | string | n8n automático | ID único generado con timestamp | `inst_1716300000000` |
| `avatar_id` | string | n8n automático | A qué avatar aplica | `sofia_01` |
| `fecha` | string | n8n automático | Fecha de creación | `2026-05-21` |
| `hora` | string | n8n automático | Fecha y hora completa | `2026-05-21 14:30` |
| `instruccion` | string | n8n (texto del admin) | El texto completo de la instrucción | `Cuando hables de outfits menciona piezas vintage` |
| `tema` | string | Groq automático | Categoría clasificada por IA | `contenido`, `tono`, `formato`, `restriccion`, `imagen` |
| `activa` | string | n8n automático / admin vía comando | Si se usa al generar posts | `SI` o `NO` |
| `prioridad` | string | n8n automático | Nivel de importancia | `alta`, `media`, `baja` |
| `origen` | string | n8n automático | Origen de la instrucción | `admin_telegram` |

**Valores posibles para `tema` (clasificados automáticamente por Groq):**
- `tono` — instrucciones sobre cómo hablar (formal, casual, humor, etc.)
- `contenido` — instrucciones sobre qué temas tocar o evitar
- `formato` — instrucciones sobre longitud, hashtags, estructura del post
- `restriccion` — cosas que el avatar nunca debe hacer o decir
- `imagen` — instrucciones sobre cómo debe verse la imagen generada (ángulo, outfit, escena, etc.)

---

### Tabla: `historial_posts`

Crece con cada post generado por `/post`.

| Columna | Tipo | Quién la llena | Descripción | Ejemplo |
|---|---|---|---|---|
| `post_id` | string | n8n automático | ID único del post | `post_1716300000000` |
| `avatar_id` | string | n8n automático | A qué avatar pertenece | `sofia_01` |
| `fecha_creacion` | string | n8n automático | Cuándo se generó | `2026-05-21 14:30` |
| `fecha_publicacion` | string | n8n automático (futuro) | Cuándo se publicó realmente | `2026-05-21 18:00` |
| `red_social` | string | n8n automático | Red para la que fue generado | `instagram` |
| `tipo` | string | n8n automático | Tipo de contenido | `post`, `historia`, `reel` |
| `texto` | string | Groq automático | Texto completo del post generado | `Hoy me desperté con ganas de...` |
| `imagen_url` | string | n8n automático | URL de la imagen generada por Pixazo | `https://pub-xxx.r2.dev/...` |
| `prompt_img` | string | n8n automático | Prompt exacto usado para generar la imagen | `realistic photo of a 24 year old...` |
| `estado` | string | n8n automático | Estado actual del post | `borrador`, `aprobado`, `publicado` |
| `instrucciones_usadas` | string | n8n automático | IDs de instrucciones activas al momento de generar | `inst_111, inst_222` |
| `notas_admin` | string | n8n automático (futuro) | Feedback del admin sobre el post | `muy bueno, repetir estilo` |

---

### Tabla: `calendario`

Planificación de contenido futuro.

| Columna | Tipo | Quién la llena | Descripción | Ejemplo |
|---|---|---|---|---|
| `avatar_id` | string | n8n automático | A qué avatar pertenece | `sofia_01` |
| `fecha_programada` | string | n8n automático | Cuándo debe publicarse | `2026-05-22 12:00` |
| `red_social` | string | n8n automático | Red social destino | `instagram` |
| `tipo_contenido` | string | n8n automático | Tipo de publicación | `post`, `historia`, `reel` |
| `tema_sugerido` | string | n8n automático | Tema propuesto para el post | `outfit del día` |
| `estado` | string | n8n automático | Estado de la entrada | `pendiente`, `generado`, `publicado` |
| `post_id` | string | n8n automático | Se llena cuando el post es generado | `post_1716300000000` |

---

### Tabla: `publicidad` *(nueva)*

Una fila por campaña/marca. El admin la edita manualmente para cargar los datos visuales y creativos de cada banner. El comando `/publicar [brand_name]` busca la fila cuyo `BRAND_NAME` coincida y lanza la generación.

| Columna | Tipo | Quién la llena | Descripción | Ejemplo |
|---|---|---|---|---|
| `BRAND_NAME` | string | Admin manual | Nombre de la marca (se usa como clave de búsqueda) | `BigCola` |
| `PRODUCT_NAME` | string | Admin manual | Nombre y descripción del producto a mostrar | `BigCola soda bottle` |
| `MODEL_DESCRIPTION` | string | Admin manual | Descripción del modelo/persona para el banner | `stylish young latina woman` |
| `MODEL_ATTITUDE` | string | Admin manual | Actitud o personalidad del modelo | `confident, energetic, urban` |
| `BACKGROUND_STYLE` | string | Admin manual | Estilo del fondo del banner | `urban graffiti wall` |
| `PRIMARY_COLORS` | string | Admin manual | Colores principales de la paleta de marca | `red, white, black` |
| `MOOD` | string | Admin manual | Atmósfera general de la imagen | `rebellious, youthful, street-style` |
| `MODEL_POSITION` | string | Admin manual | Posición del modelo en la composición | `slightly off-center` |
| `MODEL_ACTION` | string | Admin manual | Acción que realiza el modelo | `drinking the beverage` |
| `SLOGAN` | string | Admin manual | Texto/slogan principal del banner | `PIENSA EN GRANDE` |
| `ASPECT_RATIO` | string | Admin manual | Relación de aspecto de la imagen final | `4:5` |
| `RESOLUTION` | string | Admin manual | Resolución objetivo | `2400x3000` |
| `CAMERA_ANGLE` | string | Admin manual | Ángulo de cámara deseado | `low angle portrait` |
| `LIGHTING_STYLE` | string | Admin manual | Estilo de iluminación | `cinematic urban lighting` |
| `TYPOGRAPHY_STYLE` | string | Admin manual | Estilo tipográfico del banner | `bold graffiti typography` |
| `ENERGY_LEVEL` | string | Admin manual | Nivel de energía visual de la composición | `extreme` |
| `TARGET_AUDIENCE` | string | Admin manual | Audiencia objetivo de la campaña | `Gen Z urban audience` |
| `COLLAGE_INTENSITY` | string | Admin manual | Intensidad del estilo collage | `chaotic` |
| `MODEL_IMAGE_DRIVE_LINK` | string | Admin manual | URL pública de Google Drive con la imagen del modelo | `https://drive.google.com/...` |
| `PRODUCT_IMAGE_DRIVE_LINK` | string | Admin manual | URL pública de Google Drive con la imagen del producto | `https://drive.google.com/...` |
| `LOGO_IMAGE_DRIVE_LINK` | string | Admin manual | URL pública de Google Drive con el logo de la marca | `https://drive.google.com/...` |
| `STYLE_REFERENCE_DRIVE_LINK` | string | Admin manual | URL pública de Google Drive con imagen de referencia de estilo | `https://drive.google.com/...` |

**Notas importantes:**
- Las URLs de Drive deben ser de acceso público (compartidas como "cualquier persona con el enlace puede ver")
- n8n descarga cada imagen via HTTP y las convierte a base64 para enviarlas a la API de OpenAI
- Si algún link de Drive está vacío, ese slot de imagen simplemente se omite del request a OpenAI
- Se pueden tener múltiples filas (una por marca). `/publicar BigCola` busca exactamente `BigCola` en la columna `BRAND_NAME`

---

## Google Drive — estructura de carpetas

```
📁 Avatar Influencer IA/
└── 📁 sofia_01/
    ├── 📁 frente/
    ├── 📁 perfil_izquierdo/
    ├── 📁 perfil_derecho/
    ├── 📁 espalda/
    ├── 📁 casual/
    ├── 📁 outfit/
    └── 📁 expresiones/

📁 Publicidad/
└── 📁 [brand_name]/        (ej: BigCola/)
    ├── model.jpg            (foto del modelo)
    ├── product.jpg          (foto del producto)
    ├── logo.png             (logo de la marca)
    └── style_ref.jpg        (referencia de estilo visual)
```

---

## Flujo de nodos en n8n (v4)

```
Telegram Trigger
  → Extraer datos del mensaje
  → Detectar comando (Code node)
  → Router de comandos (Switch v2)
      ├── [0] instruccion     → Leer config → Extraer avatar activo
      │                         → Preparar instruccion → Clasificar tema con Groq
      │                         → Procesar tema → Guardar en Sheets → Confirmar al admin
      │
      ├── [1] instruccion_off → Leer config → Extraer avatar activo
      │                         → Desactivar en Sheets → Leer instruccion → Confirmar al admin
      │
      ├── [2] post            → Leer config → Extraer avatar activo
      │                         → Leer perfil_avatar → Leer instrucciones activas → Leer ultimos posts
      │                         → Construir prompt del agente → Llamar a Groq LLM → Extraer texto generado
      │                         → Guardar borrador en Sheets (con instrucciones_usadas)
      │                         → Construir prompt de imagen (usa perfil + instrucciones tema=imagen)
      │                         → Generar prompt de imagen (Groq) → Extraer prompt de imagen
      │                         → Llamar a Pixazo Flux → Guardar link img + prompt_img en Sheets
      │                         → Descargar imagen → Enviar imagen al admin → Enviar texto al admin
      │
      ├── [3] historial       → Leer historial_posts → Formatear → Enviar al admin
      │
      ├── [4] ayuda           → Enviar menu de comandos
      │
      ├── [5] avatar          → Actualizar config (avatar_activo) → Confirmar al admin
      │
      ├── [6] mensaje_libre   → Groq interpreta y sugiere comando → Enviar respuesta
      │
      └── [7] publicar        → Leer hoja publicidad (filtrar por BRAND_NAME)
                                → Extraer datos de campaña (Code node)
                                → Descargar imagen modelo (HTTP Request)
                                → Descargar imagen producto (HTTP Request)
                                → Descargar imagen logo (HTTP Request)
                                → Construir prompt OpenAI (Code node — ensambla el mega-prompt con todos los campos)
                                → Llamar a OpenAI gpt-image-1 (HTTP Request — multipart con imágenes base64)
                                → Extraer URL del banner generado (Code node)
                                → Descargar banner (HTTP Request)
                                → Enviar banner al admin (Telegram sendPhoto)
                                → Enviar confirmación al admin (Telegram sendMessage)
```

---

## Notas de implementación

- El `avatar_id` activo se lee de la hoja `config` al inicio de cada flujo (clave `avatar_activo`). Se cambia con el comando `/avatar [id]`.
- Los nodos `Leer config avatar activo` y `Extraer avatar activo` se ejecutan para los comandos `instruccion`, `instruccion_off` y `post`. El Router conecta las salidas [0], [1] y [2] al mismo nodo `Leer config avatar activo`.
- El campo `tema` de instrucciones acepta ahora 5 valores: `tono`, `contenido`, `formato`, `restriccion`, `imagen`. Las instrucciones de `imagen` se usan exclusivamente en el nodo `Construir prompt de imagen` y no en el prompt de texto.
- El nodo `Construir prompt del agente` filtra instrucciones excluyendo `tema = imagen`.
- El nodo `Construir prompt de imagen` usa `perfil.img_prompt_base`, `perfil.img_prompt_estilo` y `perfil.img_prompt_negativo` del avatar activo, más las instrucciones con `tema = imagen`.
- El campo `prompt_img` en `historial_posts` guarda el prompt exacto usado, útil para auditoría y mejora iterativa.
- La generación de imágenes de avatar usa Pixazo API (flux-1-schnell) con autenticación `Ocp-Apim-Subscription-Key`. La respuesta devuelve `[{ output: "url" }]`.
- El nodo `Enviar ayuda` usa `Parse Mode = None` para evitar errores de entidades en Telegram.
- Para soportar múltiples avatares: agregar nueva fila en `perfil_avatar` y usar `/avatar [nuevo_id]`.
- **Rama `/publicar`:** El comando acepta el `BRAND_NAME` exacto como argumento (ej: `/publicar BigCola`). n8n busca la fila en la hoja `publicidad`, descarga las 3 imágenes de Drive (modelo, producto, logo) como binario, las convierte a base64 y las envía junto al prompt ensamblado a la API de OpenAI (`gpt-image-1`, endpoint `/v1/images/edits` con `multipart/form-data`). La respuesta contiene la imagen generada en base64, que se decodifica y se envía directamente al admin por Telegram. Si algún link de imagen está vacío, ese campo se omite del request.
- La variable de entorno `OPENAI_API_KEY` es obligatoria para la rama `/publicar`. Agregarla en n8n Settings → Variables antes de activar el flujo.

---

## Roadmap de fases

| Fase | Estado | Descripción |
|---|---|---|
| Fase 1 | ✅ Completada | Flujo base: comandos Telegram → generación de texto → Sheets |
| Fase 2A | ✅ Completada | Nuevas columnas en Sheets: `img_prompt_*`, tabla `config`, `prompt_img` en historial |
| Fase 2B | ✅ Completada | Comando `/avatar` para cambiar avatar activo |
| Fase 2C | ✅ Completada | `avatar_id` dinámico en todo el flujo |
| Fase 2D | ✅ Completada | Prompt de imagen dinámico desde perfil + instrucciones tema=imagen |
| Fase 2E | ✅ Completada | Comando `/publicar` con generación de banners via OpenAI gpt-image-1 + imágenes de Drive |
| Fase 3 | 🔜 Pendiente | Comandos `/aprobar` y `/rechazar` con flujo de aprobación completo |
| Fase 4 | 🔜 Pendiente | Publicación automática en redes sociales (Meta Graph API / Buffer) |
| Fase 5 | 🔜 Pendiente | Mejora de consistencia visual: image-to-image con foto de referencia de Drive |
