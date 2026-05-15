# GastosApp — MQ (More Quality Events)

Aplicación web interna para registrar gastos de empresa. SPA (Single Page App) en HTML/CSS/JS puro, sin frameworks ni dependencias de build. Los datos se guardan directamente en Google Sheets vía Google Apps Script.

**URL de producción:** https://gastos-mq.vercel.app  
**Repositorio:** https://github.com/nilgf93-cmyk/Gastos_MQ

---

## Arquitectura

```
Usuario (móvil/web)
  └── index.html  (todo el frontend: HTML + CSS + JS inline)
        └── fetch POST  →  Google Apps Script Web App
                                └── Google Sheets (4 hojas)
                                └── Google Drive (fotos de tickets)
```

**Por qué no hay servidor:** el fetch usa `mode: 'no-cors'`, lo que significa que el navegador no puede leer la respuesta — pero el POST llega igualmente. Apps Script escribe en Sheets y devuelve 200 sin que el cliente lo vea. Es una limitación aceptada: si hay error en el servidor no se puede detectar desde el cliente.

**Por qué `Content-Type: text/plain`:** Google Apps Script en modo `no-cors` rechaza `application/json` con un preflight CORS bloqueado. Enviando `text/plain` el navegador lo trata como "simple request" y lo deja pasar. El body sigue siendo JSON válido, Apps Script lo parsea con `JSON.parse(e.postData.contents)`.

---

## Estructura de archivos

```
GastosApp/
├── index.html          ← toda la app (CSS + HTML + JS inline)
├── apps-script.txt     ← código del backend (Google Apps Script)
├── bg1.jpeg            ← fondo pantalla Home
├── bg2.jpeg            ← fondo pantalla Menú
├── bg3.jpeg            ← fondo pantalla Formulario
├── bg5.jpeg            ← fondo pantalla Éxito
└── README.md           ← este archivo
```

> `bg4.jpeg` existe en el repo pero no se usa actualmente.  
> `apps-script.txt` **no se despliega en Vercel** (`.vercelignore`), es solo documentación del backend.

---

## Google Sheets — estructura de columnas

| Hoja | Columnas |
|---|---|
| **Kilometraje** | Timestamp · Empleado · Fecha · Origen · Destino · Km · Tipo · Tarifa €/km · Total € · Notas · RC |
| **Dietas** | Timestamp · Empleado · Fecha · Categoría · Importe € · Descripción · URL Foto · RC |
| **Alojamiento** | Timestamp · Empleado · Fecha · Hotel · Importe € · Descripción · URL Foto · RC |
| **Otros** | Timestamp · Empleado · Fecha · Concepto · Importe € · Descripción · URL Foto · RC |

**RC** = Referencia de proyecto. Puede ser un código de 4 dígitos (ej. `1042`) o texto libre prefijado con `Visita Técnica:` (ej. `Visita Técnica: Sala Apolo`).

---

## Variables clave en `index.html`

```javascript
// URL del Web App de Apps Script
const APPS_SCRIPT_URL = 'https://script.google.com/macros/s/...../exec';

// Tarifas de kilometraje (€/km)
const TARIFAS = { evento: 0.29, normal: 0.26 };

// Dirección y coordenadas del origen por defecto (sede MQ)
const ORIGIN_ADDRESS = 'Polígono Industrial Can Vilapou, Camí de Vilapou, 3, NAVE 4, 08640 Olesa de Montserrat, Barcelona';
const ORIGIN_LAT = 41.536343;
const ORIGIN_LON = 1.900152;
```

Si hay que cambiar la URL del Apps Script (por ejemplo tras crear un nuevo despliegue), **solo hay que editar esa línea**.

### Cálculo automático de kilómetros
El formulario de Kilometraje calcula los km automáticamente al salir del campo Destino:
1. **Nominatim (OpenStreetMap)** — geocodifica el texto de destino a coordenadas lat/lon
2. **OSRM** — calcula la distancia en carretera entre origen y destino

Ambos servicios son gratuitos y sin API key. Si el usuario edita el campo Origen, también se geocodifica. Si el cálculo falla (sin conexión, dirección no encontrada), el campo km queda editable para introducir manualmente.

---

## Cómo actualizar la app

### Cambiar algo en el frontend (index.html)
1. Editar `index.html` en local
2. Subir el archivo a GitHub: repo → `index.html` → editar o subir nueva versión
3. Vercel redespliega automáticamente en ~30 segundos

### Cambiar algo en el backend (Apps Script)
1. Ir a [script.google.com](https://script.google.com) → abrir el proyecto
2. Pegar el contenido actualizado de `apps-script.txt`
3. **Implementar → Gestionar implementaciones → editar (lápiz) → Nueva versión → Implementar**
4. Si la URL del Web App cambia (raro), actualizar `APPS_SCRIPT_URL` en `index.html` y subir a GitHub

> ⚠️ Si pegas el código desde un chat o documento con formato markdown, revisa estas líneas manualmente — los links automáticos corrompen el código:
> - `data.km` (no `[data.km](http://data.km)`)
> - `data.total`
> - `folders.next()`

### Añadir un nuevo empleado
No hay lista de empleados hardcodeada. Cada usuario escribe su nombre libremente en la pantalla de inicio. El nombre se guarda en `localStorage` para la próxima vez.

---

## Despliegue desde cero

### Vercel (frontend)
1. Tener todos los archivos en un repo de GitHub
2. vercel.com → New Project → Import `Gastos_MQ`
3. Framework: **Other** — Build Command: vacío — Output Directory: `.`
4. Deploy → URL lista

### Apps Script (backend)
1. Abrir Google Sheets → Extensiones → Apps Script
2. Pegar el contenido de `apps-script.txt`
3. Ejecutar `initSheets()` una vez (crea las 4 hojas con cabeceras)
4. Implementar → Nueva implementación → Aplicación web
   - Ejecutar como: **Yo**
   - Quién tiene acceso: **Cualquier persona**
5. Copiar la URL y pegarla en `APPS_SCRIPT_URL` de `index.html`

---

## Problemas frecuentes y soluciones

### Los datos no llegan a Google Sheets
**Causa más común:** Apps Script tiene desplegada una versión antigua del código.  
**Solución:** Implementar → Gestionar implementaciones → editar → **Nueva versión** → Implementar.

**Causa alternativa:** `Content-Type` cambiado a `application/json` en el fetch.  
**Solución:** Debe ser `'Content-Type': 'text/plain'` — no cambiar esto.

### El campo RC no llega a Sheets (columna vacía)
**Causa:** El código de Apps Script no incluye `data.rc || ''` en el `appendRow`.  
**Solución:** Verificar que las 4 funciones handler (`handleKilometraje`, `handleDietas`, `handleAlojamiento`, `handleOtros`) terminan con `data.rc || ''` como último elemento del array. Re-desplegar tras corregir.

### Las imágenes de fondo no se ven
**Causa:** Los archivos `bg1.jpeg` – `bg5.jpeg` no están en la misma carpeta que `index.html`.  
**Solución:** Verificar que todos los archivos están en el repo de GitHub. La ruta en el CSS es relativa (`url('bg1.jpeg')`).

### El botón CONTINUAR aparece deshabilitado al recargar
**Comportamiento normal:** si el nombre guardado en `localStorage` tiene menos de 2 caracteres o es un valor antiguo (`Empleado 1`, etc.), se ignora y hay que escribir el nombre de nuevo.

### La app salta directamente al menú sin pedir nombre
**Comportamiento normal:** si hay un nombre guardado en `localStorage` de una sesión anterior, la app lo restaura y va directamente al menú. Para cambiar de empleado: botón "SALIR" → pantalla de inicio → escribir nuevo nombre.

### Foto no se sube / URL Foto vacía en Sheets
**Causa:** La foto supera el límite de tiempo de ejecución de Apps Script, o el base64 es demasiado grande.  
**Solución:** La app comprime las imágenes a máx. 1200px y calidad 0.75 antes de enviar. Si sigue fallando, reducir `maxSizePx` en la función `compressImage()` de `index.html`.

---

## Stack técnico

| Capa | Tecnología |
|---|---|
| Frontend | HTML + CSS + Vanilla JS (sin frameworks) |
| Hosting | Vercel (static) |
| Backend | Google Apps Script Web App |
| Base de datos | Google Sheets |
| Almacenamiento fotos | Google Drive (carpeta `Gastos_Fotos_MQ`) |
| Fuentes | Inter + Inter Tight (Google Fonts) |

---

## Contacto / historial
App desarrollada en abril 2026 para uso interno de More Quality Events.  
Para cualquier duda técnica, este README junto con `index.html` y `apps-script.txt` contienen toda la información necesaria para entender y modificar la app.
