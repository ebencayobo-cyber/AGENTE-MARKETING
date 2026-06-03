# Fase 2F — Migración de `/post` a OpenAI gpt-image-1 (workflow v6)

Guía de implementación que corresponde al archivo `Avatar_Influencer_IA___Sistema_Completo_v6.json` ya generado. **Solo afecta la rama `/post`** (más una corrección puntual en `/publicar`). El resto del flujo queda intacto.

> Si importas el v6 directamente, los nodos y conexiones ya están aplicados. Esta guía documenta qué cambió y qué debes completar manualmente en n8n.

---

## Resumen de cambios aplicados en v6

1. **Eliminado** `Llamar a Pixazo Flux`.
2. **Eliminado** `Descargar imagen` (ya no se redescarga la imagen desde Drive).
3. **8 nodos nuevos**: 7 de generación con OpenAI + `Recuperar binario imagen`.
4. La imagen se envía a Telegram desde el **binario en memoria**, no desde la URL de Drive.
5. El registro en `historial_posts` ocurre **antes** del envío al admin.
6. **Corregido** `quality: "hight"` -> `"high"` en `OpenAI Generar banner` (rama `/publicar`).

---

## Cadena final de la rama `/post`

```
Extraer prompt de imagen
  -> Preparar items de referencia    (valida las 3 fotos, arma 3 items)
  -> HTTP Descargar referencia       (baja las 3 fotos de Drive)
  -> Juntar referencias              (aggregate -> data, data_1, data_2)
  -> OpenAI Generar imagen avatar    (gpt-image-1, /v1/images/edits, quality high)
  -> Decodificar imagen              (b64 -> binario en memoria)
  -> Subir a Drive                   (carpeta generadas/)
  -> Construir URL publica
  -> Guardar link img                (registra en historial_posts PRIMERO)
  -> Recuperar binario imagen        (reinyecta el binario que Sheets descarto)
  -> Enviar imagen                   (envia el binario; NO redescarga)
  -> Enviar texto al admin
```

---

## Pasos manuales en n8n tras importar el v6

### Paso 0 — Datos en Google Sheets
En la hoja `perfil_avatar`, agregar 3 columnas y rellenarlas con los enlaces **publicos** de Drive de las 3 fotos del avatar:
- `img_ref_frente_link` (foto frontal de la cara)
- `img_ref_perfil_link` (perfil de la cara)
- `img_ref_cuerpo_link` (medio cuerpo)

> Se exigen las 3. Si falta alguna, el nodo `Preparar items de referencia` lanza un error controlado.

### Paso 1 — Carpeta de salida en Drive
1. Crear (si no existe) la carpeta `generadas/` dentro de la carpeta del avatar.
2. Compartirla como "cualquier persona con el enlace" (para que `imagen_url` sea publica).
3. Copiar su **folder ID** (lo que va en la URL despues de `/folders/`).
4. Pegarlo en el nodo `Preparar items de referencia`, reemplazando `PEGAR_FOLDER_ID_DE_GENERADAS`.

### Paso 2 — Credencial de Google Drive
El workflow v5 **no tenia ningun nodo de Drive**, asi que no hay credencial reutilizable. Abrir el nodo `Subir a Drive` y conectar tu cuenta de Google Drive (OAuth, un clic).

### Paso 3 — Verificar variable
Confirmar que `OPENAI_API_KEY` existe en Settings -> Variables (ya la usa `/publicar`). No se anade ninguna variable nueva.

---

## Detalle de los nodos clave (referencia)

### `Preparar items de referencia` (Code)
Convierte los 3 links de Drive a URL de descarga directa, valida que existan los 3, y emite un item por foto arrastrando `prompt_imagen`, `post_id`, `chat_id` y el folder ID.

### `OpenAI Generar imagen avatar` (HTTP)
`POST https://api.openai.com/v1/images/edits`, multipart-form-data, `Authorization: Bearer {{ $vars.OPENAI_API_KEY }}`.
Campos: 3x `image[]` (`data`, `data_1`, `data_2`), `model=gpt-image-1`, `prompt={{ prompt_imagen }}`, `n=1`, `size=1024x1536`, `output_format=png`, `quality=high`, `timeout=120000`.

### `Decodificar imagen` (Code)
`b64_json` -> `prepareBinaryData` -> `binary.data`. El binario queda disponible en memoria para reusarlo despues.

### `Recuperar binario imagen` (Code)
Los nodos de Sheets/Code no arrastran binarios, asi que el binario se pierde tras `Guardar link img`. Este nodo lo recupera por referencia:
```js
const dec = $('Decodificar imagen').item;
const tg  = $('Extraer texto generado').item.json;
return [{
  json: { chat_id: tg.chat_id, post_id: tg.post_id },
  binary: { data: dec.binary.data }
}];
```
Asi `Enviar imagen` recibe el binario y el orden guardar->enviar se mantiene.

---

## Correccion en `/publicar`
En `OpenAI Generar banner`, el campo `quality` estaba como `"hight"` (typo). Corregido a `"high"`. Sin otros cambios en esa rama.

---

## Pruebas

1. `/post instagram outfit de verano` -> verificar que llega la imagen al Telegram y que la cara coincide con las fotos de referencia.
2. Revisar en `historial_posts` que `imagen_url` (Drive) y `prompt_img` se guardaron, y que la fila existe **antes** de que llegue el mensaje.
3. Abrir el `imagen_url` en incognito para confirmar que es publico.

## Checklist
- [ ] 3 columnas nuevas en `perfil_avatar` con links validos y publicos
- [ ] Carpeta `generadas/` creada, publica, y su folder ID pegado en `Preparar items de referencia`
- [ ] Credencial de Google Drive conectada en `Subir a Drive`
- [ ] `OPENAI_API_KEY` presente en Variables
- [ ] Prueba end-to-end OK (imagen + texto + fila en Sheets)
