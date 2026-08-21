# InvitationGoogleSite

Invitación digital para la Primera Comunión de **Ashley Álvarez Lozano**, publicada como Web App con **Google Apps Script**. Cada invitado recibe un link único (`?id=CODIGO`) que muestra su nombre personalizado y le permite confirmar asistencia con un solo clic; la confirmación se guarda directamente en una **Google Sheet**.

## ¿De qué trata?

- `Codigo.gs`: backend en Apps Script.
  - `doGet(e)`: lee el parámetro `id` de la URL, busca al invitado en la hoja de cálculo y renderiza `Index.html` con sus datos (`codigo`, `nombre`, `maximo`). Si falta el `id` o no se encuentra el código, muestra un mensaje de error simple en vez de la invitación.
  - `buscarInvitado(codigo)`: recorre la hoja `Invitados` y devuelve la fila que coincide con el código (comparación insensible a mayúsculas/espacios).
  - `confirmarAsistencia(codigo)`: se llama desde el frontend (`google.script.run`) cuando el invitado toca "Sí, confirmar"; busca la fila por código y escribe `"Si"` en la columna `Confirmados`.
  - `ENCODE_WHATSAPP(texto)`: helper para codificar texto para links de WhatsApp (`wa.me`/`api.whatsapp.com`). Está definido pero **no se usa actualmente** en la plantilla — queda disponible para si se agrega un botón de "compartir por WhatsApp" a futuro.
- `Index.html`: la plantilla de la invitación (HTML Service con scriptlets `<?= ... ?>`), con:
  - Sección de portada con el nombre del invitado (`<?= nombre ?>`).
  - Datos del evento: ceremonia religiosa y recepción, con fecha objetivo `2026-11-15T15:00:00` usada también para el countdown en JS.
  - Fecha límite de confirmación (actualmente "10 de noviembre de 2026", texto fijo en el HTML).
  - Modal de confirmación de asistencia que invoca `confirmarAsistencia(codigo)` vía `google.script.run`.

## Configuración necesaria en Google Sheets

1. **Crear/usar una hoja de cálculo de Google** y copiar su ID (el valor entre `/d/` y `/edit` en la URL) en `Codigo.gs`:
   ```js
   const ID_HOJA_CALCULO = "10D53UC6tnjjzcio7e44JCjtLshT5ITWKEApbuZUxl3s";
   ```
2. **Crear una pestaña (tab) llamada exactamente `Invitados`** (definido en `NOMBRE_HOJA`). El nombre es sensible a mayúsculas/minúsculas y debe coincidir exactamente.
3. **La primera fila debe ser el encabezado**, y las columnas deben respetar este orden (columnas A, B y C se leen por posición, no por nombre):

   | Columna | Encabezado sugerido | Uso |
   |---|---|---|
   | A | Código | Código único del invitado, usado en la URL como `?id=CODIGO` |
   | B | Nombre | Nombre que se muestra en la invitación |
   | C | Máximo | Cantidad máxima de acompañantes permitidos (leído pero aún no mostrado en el HTML) |
   | (una más) | **Confirmados** | Debe existir una columna con el encabezado exacto `Confirmados` (no importa en qué posición). Ahí se escribe `"Si"` cuando el invitado confirma. |

   > `confirmarAsistencia` busca la columna `Confirmados` **por nombre de encabezado** (insensible a mayúsculas), así que puede ir en cualquier posición siempre que el encabezado diga exactamente `Confirmados`.

4. **Los códigos de invitado (columna A) deben ser únicos** — la búsqueda usa el primer código que coincida.
5. **Permisos**: la hoja debe ser accesible por la cuenta de Google que despliega el Apps Script (lo más simple es que ambas —hoja y proyecto de Apps Script— pertenezcan a la misma cuenta de Google).

### Ejemplo de cómo debe quedar la hoja `Invitados`

| | A: Código | B: Nombre | C: Máximo | D: Confirmados |
|---|---|---|---|---|
| 1 | Código | Nombre | Máximo | Confirmados |
| 2 | ABC123 | Familia Pérez Gómez | 4 | |
| 3 | XYZ789 | Juan Rodríguez | 2 | Si |
| 4 | LMN456 | María Torres | 1 | |

- La fila 1 es el encabezado (obligatoria).
- La columna `Confirmados` empieza **vacía**; Apps Script escribe `Si` ahí solo cuando el invitado confirma desde la invitación.
- El link de este ejemplo para la fila 2 sería `...?id=ABC123`.

## Paso a paso para cargar el código en Apps Script

1. **Abrí la Google Sheet** que vas a usar como base de datos de invitados (la que copiaste en `ID_HOJA_CALCULO`).
2. En el menú superior andá a **Extensiones → Apps Script**. Esto abre un proyecto de Apps Script ya vinculado a esa hoja (así no hace falta compartir permisos aparte).
3. Por defecto el proyecto trae un archivo `Código.gs` (o `Code.gs`) vacío:
   - Borrá el contenido que trae por defecto y pegá ahí **todo** el contenido de `Codigo.gs` de este repo.
   - Renombralo (ícono de los 3 puntos → **Cambiar nombre**) para que quede `Codigo` (o dejalo como `Code`, el nombre del archivo `.gs` no importa, Apps Script solo necesita que las funciones existan).
4. Agregá el archivo del HTML:
   - Click en el **+** al lado de "Archivos" → **HTML**.
   - Nombralo exactamente **`Index`** (sin `.html`, Apps Script lo agrega solo). El nombre importa porque `Codigo.gs` hace `HtmlService.createTemplateFromFile("Index")`.
   - Pegá ahí todo el contenido de `Index.html` de este repo.
5. Verificá el ID de la hoja: en `Codigo.gs`, confirmá que `ID_HOJA_CALCULO` tenga el ID correcto de tu Google Sheet (si abriste el editor desde **Extensiones → Apps Script** de la propia hoja, podés incluso reemplazarlo por `SpreadsheetApp.getActiveSpreadsheet().getId()`, aunque no es obligatorio).
6. Guardá el proyecto (ícono de disquete o `Ctrl+S`).
7. (Opcional pero recomendable) Ejecutá una vez la función `buscarInvitado` manualmente desde el editor (seleccionándola arriba y dándole ▶ Ejecutar) para que Google te pida autorizar los permisos de acceso a la hoja antes de publicar la Web App.

## Despliegue como Web App

1. Con `Codigo.gs` e `Index.html` ya cargados en el proyecto de Apps Script (paso anterior).
2. **Implementar → Nueva implementación → seleccionar tipo: Aplicación web**.
   - Ejecutar como: **Yo** (el dueño del script, para que tenga permiso de escribir en la hoja).
   - Quién tiene acceso: según se necesite (por ejemplo, "Cualquier persona").
3. Autorizá los permisos que pida Google (acceso a la hoja de cálculo) la primera vez.
4. Compartir el link generado agregando `?id=CODIGO` por cada invitado, por ejemplo:
   `https://script.google.com/macros/s/XXXXX/exec?id=ABC123`
5. Cada vez que se cambien `Codigo.gs` o `Index.html` hay que crear una **nueva versión de implementación** (Implementar → Administrar implementaciones → editar → Nueva versión) para que los cambios se reflejen en el link publicado.

## Cosas a tener en cuenta

- Si un invitado abre el link sin `?id=` o con un código que no existe en la hoja, ve una página de error en vez de la invitación.
- El campo `maximo` (columna C) ya se recibe en el backend pero todavía no se usa/muestra en `Index.html` — si se quiere limitar/mostrar la cantidad de acompañantes, hay que agregarlo a la plantilla.
- La fecha del evento (`2026-11-15T15:00:00`) y la fecha límite de confirmación (10 de noviembre de 2026) están hardcodeadas en `Index.html`; si cambia la fecha del evento hay que actualizarlas ahí manualmente.
