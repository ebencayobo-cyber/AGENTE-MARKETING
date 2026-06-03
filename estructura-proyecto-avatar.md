# Estructura oficial del proyecto Avatar Influencer IA
> Documento de referencia. Toda decisión de diseño está aquí.
> Última actualización: 2026-06-03 (workflow v6.2 — botón Generar Post pide red + tema)

---

## Stack tecnológico

| Componente | Herramienta |
|---|---|
| Orquestador | n8n Cloud 2.21.6 |
| Canal de administración | Telegram Bot |
| LLM texto | Groq API — modelo `llama-3.3-70b-versatile` |
| Generación de imágenes avatares | **OpenAI API — modelo `gpt-image-1` (endpoint `/v1/images/edits`)** *(migrado desde Pixazo, ver Fase 2F)* |
| Generación de banners publicitarios | OpenAI API — modelo `gpt-image-1` |
| Tendencias TikTok | Groq API (análisis sobre datos recolectados) |
| Tendencias YouTube | YouTube Data API v3 |
| Base de datos | Google Sheets |
| Almacenamiento de imágenes | Google Drive |
| Variables de entorno | n8n Settings → Variables |

---

## Variables de entorno en n8n

| Variable | Descripción |
|---|---|
| `GROQ_API_KEY` | API Key de Groq |
| `GOOGLE_SHEET_ID` | ID del Google Sheet principal |
| `OPENAI_API_KEY` | API Key de OpenAI (banners publicitarios **y** imágenes de avatar) |
| `TELEGRAM_BOT_TOKEN` | Token del bot de Telegram (usado en nodos Code que llaman a la API directa) |
| `YOUTUBE_API_KEY` | API Key de YouTube Data API v3 (comando `/tendencias_yt`) |
| `USER_ID_TELEGRAM` | ID de Telegram del admin |
| ~~`PIXAZO_API_KEY`~~ | *Obsoleta tras Fase 2F. Puede eliminarse.* |

---

## Comandos de Telegram (admin)

| Comando | Sintaxis | Descripción |
|---|---|---|
| `/avatar` | `/avatar [id]` | Cambia el avatar activo en la sesión. Se guarda en la hoja `config`. |
| `/instruccion` | `/instruccion [texto]` | Enseñarle algo al avatar activo. Se guarda en Sheets automáticamente. El tema se clasifica con IA. |
| `/instruccion_off` | `/instruccion_off [id]` | Desactiva una instrucción por su ID sin borrarla. Flujo con botones de confirmación (`offsel`/`offyes`/`offno`). |
| `/post` | `/post [red_social] [tema]` | Genera un borrador de post + imagen. Lee perfil + instrucciones activas + historial. |
| `/historial` | `/historial [n]` | Muestra los últimos N posts generados. |
| `/publicar` | `/publicar [brand_name]` | Genera un banner publicitario para la marca indicada. Lee `publicidad`, descarga imágenes de Drive y llama a OpenAI gpt-image-1. |
| `/tendencias` | `/tendencias` | Analiza tendencias de TikTok en Bolivia (últimos 7 días) y Groq sugiere ideas de contenido para el avatar. |
| `/tendencias_yt` | `/tendencias_yt` | Tendencias de YouTube Bolivia. Submenú con botones: Reels/Shorts (`ytshort`) o Videos normales (`ytlong`). Usa YouTube Data API v3. |
| `/ayuda` | `/ayuda` (o `/start`) | Muestra el menú de comandos. |

### Callbacks internos (botones inline, no se tipean)
| callback_data | Función |
|---|---|
| `offsel:[id]` | Selección de instrucción a desactivar |
| `offyes:[id]` | Confirmar desactivación |
| `offno` | Cancelar desactivación |
| `ytshort` / `ytlong` | Elegir tipo de tendencia de YouTube |
| `publicar:[brand]` | Elegir marca desde botones |
| `postred:` | Abre el submenú de redes al tocar "Generar Post" |
| `post:[red]` | Elegir red (instagram/tiktok/historia) desde botones → luego pide el tema |

---

## Flujo conversacional con estado (`estado_pendiente`)

El sistema soporta comandos en dos pasos. Si un comando requiere argumento y llega vacío, n8n:
1. Guarda `estado_pendiente = comando|extra` en la hoja `config`.
2. Pregunta al usuario (texto o botones inline).
3. Al recibir la respuesta, el nodo `Resolver intencion` lee el estado pendiente, lo combina con el nuevo mensaje y reanuda el comando.

Nodos clave: `Leer estado pendiente` → `Resolver intencion` → `Ejecutar o pedir?` (Switch) → `Guardar/Limpiar estado pendiente` → `Tipo de pregunta` → (`Enviar botones avatares` / `Enviar botones marcas` / `Leer avatar activo (off)` / `Enviar pregunta texto`) → `Restaurar contexto comando` → `Router de comandos`.

---

## Google Sheets — estructura de tablas

### Reglas generales
- Ninguna columna se rellena manualmente por el admin (salvo las indicadas).
- Todo se escribe automáticamente desde n8n.
- El admin puede editar `perfil_avatar` (datos base del personaje) y `publicidad` (datos de marca).
- Las hojas `config`, `historial_posts`, `calendario` e `instrucciones` las gestiona n8n.

---

### Tabla: `perfil_avatar`

Una fila por avatar. Se crea manualmente una sola vez al crear el avatar.

| Columna | Tipo | Descripción | Ejemplo |
|---|---|---|---|
| `avatar_id` | string | Identificador único del avatar | `sofia_01` |
| `nombre` | string | Nombre del personaje | `Sofía Reyes` |
| `edad` | number | Edad ficticia | `24` |
| `ciudad` | string | Ciudad ficticia | `Madrid` |
| `bio` | string | Descripción del perfil público | `Diseñadora freelance amante del café` |
| `tono_voz` | string | Cómo habla el avatar | `casual, cercana, con humor sutil` |
| `intereses` | string | Temas que toca | `moda, viajes, café, diseño` |
| `frases_tipicas` | string | Muletillas | `"Total", "La verdad es que..."` |
| `lo_que_nunca_dice` | string | Restricciones de contenido | `no usa groserías, no habla de política` |
| `redes_activas` | string | Redes donde publica | `instagram, tiktok` |
| `carpeta_drive` | string | URL de carpeta de imágenes en Drive | `https://drive.google.com/...` |
| `img_prompt_base` | string | Descripción física. *(v6.1: ya NO se usa en el prompt de imagen; queda como documentación/fallback)* | `24 year old spanish woman...` |
| `img_prompt_estilo` | string | Estilo visual consistente | `realistic photo, natural lighting, high quality` |
| `img_prompt_negativo` | string | *(v6.1: ya NO se usa en el prompt; irrelevante con edits+fotos)* | `cartoon, anime, ugly, deformed` |
| `img_ref_frente_link` | string | **(Fase 2F)** URL pública Drive — foto frontal de la cara | `https://drive.google.com/...` |
| `img_ref_perfil_link` | string | **(Fase 2F)** URL pública Drive — foto de perfil de la cara | `https://drive.google.com/...` |
| `img_ref_cuerpo_link` | string | **(Fase 2F)** URL pública Drive — foto de medio cuerpo | `https://drive.google.com/...` |

> Las 3 fotos de referencia son lo que da consistencia facial/corporal a las imágenes generadas con `gpt-image-1`. Si algún link está vacío, ese slot se omite del request.

---

### Tabla: `config`

| Columna | Tipo | Quién la llena | Descripción | Ejemplo |
|---|---|---|---|---|
| `clave` | string | n8n automático | Identificador de la configuración | `avatar_activo` |
| `valor` | string | n8n automático | Valor actual | `sofia_01` |
| `actualizado` | string | n8n automático | Fecha y hora del último cambio | `2026-05-22 14:30` |

**Claves usadas:** `avatar_activo`, `estado_pendiente`, y claves de cache de tendencias (TikTok / `[SHORTS]`/`[VIDEOS]` de YouTube).

**Filas iniciales requeridas:**

| clave | valor |
|---|---|
| `avatar_activo` | `sofia_01` |

---

### Tabla: `instrucciones`
Crece con cada `/instruccion`. Columnas: `id`, `avatar_id`, `tema` (`tono`/`contenido`/`formato`/`restriccion`/`imagen`), `instruccion`, `activa` (`SI`/`NO`).

---

### Tabla: `historial_posts`

| Columna | Tipo | Quién la llena | Descripción | Ejemplo |
|---|---|---|---|---|
| `post_id` | string | n8n automático | ID único del post | `post_1716300000000` |
| `avatar_id` | string | n8n automático | A qué avatar pertenece | `sofia_01` |
| `fecha_creacion` | string | n8n automático | Cuándo se generó | `2026-05-21 14:30` |
| `fecha_publicacion` | string | n8n automático (futuro) | Cuándo se publicó | `2026-05-21 18:00` |
| `red_social` | string | n8n automático | Red destino | `instagram` |
| `tipo` | string | n8n automático | Tipo de contenido | `post` |
| `texto` | string | Groq automático | Texto del post | `Hoy me desperté con ganas de...` |
| `imagen_url` | string | n8n automático | URL de la imagen (Drive tras Fase 2F) | `https://drive.google.com/...` |
| `prompt_img` | string | n8n automático | Prompt usado para la imagen | `realistic photo of a 24 year old...` |
| `estado` | string | n8n automático | Estado del post | `borrador` |
| `instrucciones_usadas` | string | n8n automático | IDs de instrucciones activas | `inst_111, inst_222` |
| `notas_admin` | string | n8n automático (futuro) | Feedback del admin | `muy bueno, repetir estilo` |

---

### Tabla: `calendario`
Planificación de contenido futuro. Columnas: `avatar_id`, `fecha_programada`, `red_social`, `tipo_contenido`, `tema_sugerido`, `estado`, `post_id`.

---

### Tabla: `publicidad`
Una fila por campaña/marca. El admin la edita manualmente. `/publicar [brand_name]` busca por `BRAND_NAME`.

Columnas: `BRAND_NAME`, `PRODUCT_NAME`, `MODEL_DESCRIPTION`, `MODEL_ATTITUDE`, `BACKGROUND_STYLE`, `PRIMARY_COLORS`, `MOOD`, `MODEL_POSITION`, `MODEL_ACTION`, `SLOGAN`, `ASPECT_RATIO`, `RESOLUTION`, `CAMERA_ANGLE`, `LIGHTING_STYLE`, `TYPOGRAPHY_STYLE`, `ENERGY_LEVEL`, `TARGET_AUDIENCE`, `COLLAGE_INTENSITY`, `MODEL_IMAGE_DRIVE_LINK`, `MODEL2_IMAGE_DRIVE_LINK`, `PRODUCT_IMAGE_DRIVE_LINK`, `LOGO_IMAGE_DRIVE_LINK`, `STYLE_REFERENCE_DRIVE_LINK`.

- Las URLs de Drive deben ser de acceso público.
- n8n descarga cada imagen y la envía como binario a OpenAI.
- Si un link está vacío, ese slot se omite del request.

---

## Google Drive — estructura de carpetas

```
📁 Avatar Influencer IA/
└── 📁 sofia_01/
    ├── 📁 frente/             (foto frontal — referencia Fase 2F)
    ├── 📁 perfil_izquierdo/   (perfil — referencia Fase 2F)
    ├── 📁 perfil_derecho/
    ├── 📁 cuerpo/             (medio cuerpo — referencia Fase 2F)
    ├── 📁 escenarios/         (paisajes/escenarios reutilizables — Fase 2G)
    ├── 📁 generadas/          (salida de /post subida automáticamente)
    ├── 📁 casual/
    ├── 📁 outfit/
    └── 📁 expresiones/

📁 Publicidad/
└── 📁 [brand_name]/
    ├── model.jpg
    ├── product.jpg
    ├── logo.png
    └── style_ref.jpg
```

---

## Cómo se genera el prompt de imagen (v6.1 — identidad por fotos)

**Principio clave:** la identidad del personaje (cara, vitíligo, marcas, cuerpo) viene **solo de las 3 fotos de referencia**, NUNCA del texto. Groq genera únicamente la escena. El texto nunca describe rasgos físicos, porque hacerlo introduce variación y deforma al personaje.

Reparto de responsabilidades:
- **3 fotos de referencia** → identidad (gpt-image-1 las copia).
- **Groq** → SOLO la escena/acción/ambiente/estilo a partir del post (ej: "on a fishing boat at golden hour, holding a rod"). Tiene prohibido describir rostro, pelo, piel o rasgos.
- **Prompt final** → una orden imperativa de identidad al principio ("MUST be identical to the attached references, preserve vitiligo/skin marks exactly, do not alter") + la escena de Groq después.

Flujo (2 llamadas a Groq, igual que antes):
1. **`Construir prompt del agente` → `Llamar a Groq LLM`** → texto del post (con `temperature: 0.9`).
2. **`Construir prompt de imagen`** pide a Groq SOLO la escena (≤50 palabras, sin rasgos físicos). **`Generar prompt de imagen`** hace la llamada. **`Extraer prompt de imagen`** antepone la orden de identidad y arma el `prompt_imagen` final.

**Qué se usa del Sheet `perfil_avatar` en el prompt de imagen:**
| Campo | En el prompt de imagen | Motivo |
|---|---|---|
| `img_ref_frente/perfil/cuerpo_link` | **Sí (identidad)** | Las fotos hacen el trabajo |
| `img_prompt_estilo` | **Sí (estética)** | Luz, tipo de foto — no es identidad |
| `img_prompt_base` | **No** (queda en Sheet como doc/fallback) | Describir la cara la deforma |
| `img_prompt_negativo` | **No** (queda en Sheet) | Irrelevante con edits+fotos |

> Por qué `/publicar` siempre funcionó bien y `/post` no: `/publicar` ya ordenaba "preserve identity from reference" y dejaba que las fotos mandaran; `/post` describía la cara con palabras desde el Sheet. v6.1 alinea `/post` con ese enfoque.

---

## Rama `/post` — subflujo de imagen (Fase 2F, OpenAI) ✅

Reemplaza la generación con Pixazo. La construcción del prompt (2 llamadas a Groq, arriba) se mantiene igual; cambia el motor de imagen, se añaden 3 fotos de referencia, y el envío a Telegram usa el binario en memoria (sin redescargar de Drive).

```
Guardar borrador en Sheets
  → Construir prompt de imagen (Code)        [sin cambios]
  → Generar prompt de imagen (Groq)          [sin cambios]
  → Extraer prompt de imagen (Code)          [sin cambios → produce prompt_imagen]
  → Preparar items de referencia (Code)      [NUEVO — exige las 3 fotos, emite 3 items]
  → HTTP Descargar referencia (HTTP, file)   [NUEVO]
  → Juntar referencias (Aggregate, binarios) [NUEVO → data, data_1, data_2]
  → OpenAI Generar imagen avatar (HTTP)       [NUEVO — /v1/images/edits multipart]
  → Decodificar imagen (Code, b64 → binary)  [NUEVO — binario queda en memoria]
  → Subir a Drive (Google Drive)              [NUEVO → carpeta generadas/]
  → Construir URL publica (Code)              [NUEVO]
  → Guardar link img (Google Sheets)         [imagen_url + prompt_img — registra ANTES del envío]
  → Recuperar binario imagen (Code)          [NUEVO — reinyecta binario perdido al pasar por Sheets]
  → Enviar imagen (Telegram, binario)        [usa binario en memoria, NO redescarga]
  → Enviar texto al admin                    [sin cambios]
```

**Nodos eliminados respecto a v5:** `Llamar a Pixazo Flux` y `Descargar imagen` (la redescarga desde Drive ya no es necesaria).

**Por qué `Recuperar binario imagen`:** los nodos de Google Sheets y Code no arrastran propiedades binarias. El binario que crea `Decodificar imagen` se pierde al pasar por `Subir a Drive → Construir URL publica → Guardar link img`. Este nodo lo recupera por referencia (`$('Decodificar imagen').item.binary.data`) y lo reinyecta en el item para que `Enviar imagen` lo mande. Así se logra el orden **registrar-en-Sheets-primero, enviar-después** sin perder el binario y sin redescargar de Drive.

**Parámetros del nodo OpenAI:** endpoint `https://api.openai.com/v1/images/edits`, `Authorization: Bearer {{ $vars.OPENAI_API_KEY }}`, `multipart-form-data`, 3 campos `image[]` fijos (`data`, `data_1`, `data_2`), `model=gpt-image-1`, `prompt={{ prompt_imagen }}`, `n=1`, `size=1024x1536`, `output_format=png`, `quality=high`, `timeout=120000`. **Exige siempre las 3 fotos**: si falta alguna, `Preparar items de referencia` lanza error.

**Pendiente al importar v6:** (1) pegar el folder ID de `generadas/` en `Preparar items de referencia`; (2) conectar credencial de Google Drive en `Subir a Drive` (el workflow no tenía ningún nodo de Drive previo); (3) carpeta `generadas/` compartida públicamente; (4) 3 columnas nuevas en `perfil_avatar`.

---

## Roadmap de fases

| Fase | Estado | Descripción |
|---|---|---|
| Fase 1 | ✅ Completada | Flujo base: comandos Telegram → generación de texto → Sheets |
| Fase 2A | ✅ Completada | Columnas `img_prompt_*`, tabla `config`, `prompt_img` en historial |
| Fase 2B | ✅ Completada | Comando `/avatar` para cambiar avatar activo |
| Fase 2C | ✅ Completada | `avatar_id` dinámico en todo el flujo |
| Fase 2D | ✅ Completada | Prompt de imagen dinámico desde perfil + instrucciones tema=imagen |
| Fase 2E | ✅ Completada | Comando `/publicar` (banners via OpenAI gpt-image-1 + imágenes de Drive) |
| Fase 2F | ✅ Completada | **`/post` migrado de Pixazo a OpenAI `gpt-image-1` (`/v1/images/edits`) con 3 fotos de referencia + subida a Drive + envío del binario en memoria (workflow v6).** |
| Fase 2G | 🔜 Pendiente | Escenario por **texto**: el usuario describe el escenario y se inyecta al prompt de imagen (1 línea de cambio). |
| Fase 2H | 🔜 Pendiente | Escenario por **imagen**: flujo conversacional para que el usuario envíe una foto por Telegram o elija una de la carpeta `escenarios/` en Drive. La imagen entra como `image[]` extra en el request a OpenAI. Requiere: recibir foto de Telegram (`message.photo[]` → `getFile` → descarga), reutilizar el sistema `estado_pendiente`, y listar carpeta de Drive con botones inline. |
| Fase 2I | ✅ Completada | Comando `/instruccion_off` con confirmación por botones (`offsel`/`offyes`/`offno`). *(parte del antiguo roadmap Fase 3)* |
| Fase 2J | ✅ Completada | Comandos `/tendencias` (TikTok Bolivia) y `/tendencias_yt` (YouTube Data API, shorts/largos). |
| Fase 3 | 🔜 Pendiente | Comandos `/aprobar` y `/rechazar` con flujo de aprobación completo |
| Fase 4 | 🔜 Pendiente | Publicación automática en redes (Meta Graph API / Buffer) |

---

## Notas de implementación

- El `avatar_id` activo se lee de la hoja `config` (clave `avatar_activo`). Se cambia con `/avatar [id]`.
- El campo `tema` de instrucciones acepta: `tono`, `contenido`, `formato`, `restriccion`, `imagen`. Las de `imagen` se usan solo en `Construir prompt de imagen`.
- El campo `prompt_img` en `historial_posts` guarda el prompt exacto, útil para auditoría.
- **Imágenes de avatar (Fase 2F, v6):** OpenAI `gpt-image-1`, endpoint `/v1/images/edits`, `multipart/form-data`. Se envían siempre 3 fotos de referencia (frente, perfil, cuerpo) como `image[]`. La respuesta trae `b64_json`, que se decodifica a binario en memoria. Ese binario se sube a Drive (para `imagen_url`) **y** se reinyecta tras el guardado en Sheets para enviarlo a Telegram sin redescargar. El registro en `historial_posts` ocurre **antes** del envío al admin (importante para que `/aprobar [post_id]` encuentre la fila).
- **Reinyección de binario:** Sheets y Code no arrastran binarios; el nodo `Recuperar binario imagen` lo recupera de `Decodificar imagen` por referencia. Esto permite el orden seguro guardar→enviar sin perder la imagen.
- **Naturalidad del texto (v6.1):** en `Construir prompt del agente`, las frases típicas se presentan como ejemplos de tono (no obligación), las instrucciones como pautas integrables (no checklist), y se añadieron requisitos anti-robótico ("escribe espontáneo, no como anuncio", "varía el inicio", "no fuerces las frases"). El nodo `Llamar a Groq LLM` usa `temperature: 0.9`. Para mejorar aún más, se puede añadir un ejemplo (one-shot) de un post bien logrado del personaje.
- **Typo corregido (v6):** en `OpenAI Generar banner` (/publicar) el campo `quality` decía `"hight"`; corregido a `"high"`. Los nodos nuevos de Fase 2F ya usan `"high"`.
- **Coste/latencia:** `gpt-image-1` es más caro y lento que Pixazo. Usar `timeout: 120000` en el nodo OpenAI.
- `OPENAI_API_KEY` ya existe (de `/publicar`); no se añade variable nueva en Fase 2F.
- El nodo `Enviar ayuda` usa `Parse Mode = None` para evitar errores de entidades en Telegram.
- Para múltiples avatares: agregar fila en `perfil_avatar` + `/avatar [nuevo_id]`.
- **Rama `/publicar`:** acepta `BRAND_NAME` exacto (ej: `/publicar BigCola`). Busca en `publicidad`, descarga las imágenes de Drive como binario, las junta (aggregate) y las envía con el prompt ensamblado a `gpt-image-1`. La respuesta en base64 se decodifica y se envía al admin por Telegram.
