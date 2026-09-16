# Panel de Resultados — Polilla Digital

Dashboard de resultados de marketing digital (SEO, AEO/GEO, Google Ads, Meta Ads
y menciones en IA) para clientes de Polilla Digital. Es una app de una sola
página (`index.html`, sin build) pensada para publicarse con GitHub Pages.

Usa dos servicios externos:
- **OAuth de Google** (Search Console API + GA4 Data API) — el navegador lee los
  datos directo, sin backend propio.
- **Cloudflare Worker** (`cloudflare-worker.js`) — proxy para la API de Anthropic
  (sección "Recomendaciones") y la Graph API de Meta (sección Meta Ads), porque
  esas dos llamadas necesitan API keys secretas que no pueden vivir en un HTML
  público.

Este repo es una copia personal, con marca y credenciales propias, de un
dashboard usado internamente en la agencia Bigbuda.

## Puesta en marcha

Antes de usar el dashboard hay que completar 3 pasos manuales. Ninguno se
puede automatizar desde el código — son configuración en Google Cloud,
Cloudflare y tu DNS.

### 1. Google Cloud — OAuth Client ID

1. Crea (o reusa) un proyecto en [Google Cloud Console](https://console.cloud.google.com/).
2. Habilita estas dos APIs (**APIs & Services → Library**):
   - **Google Search Console API**
   - **Google Analytics Data API**
3. Configura la pantalla de consentimiento OAuth (**APIs & Services → OAuth
   consent screen**) — tipo *External*, agrega tu correo como usuario de
   prueba si la app queda en modo *Testing*.
4. Crea las credenciales (**APIs & Services → Credentials → Create
   Credentials → OAuth client ID**):
   - Tipo de aplicación: **Web application**.
   - **Authorized JavaScript origins**: la URL donde vas a publicar el
     dashboard (ej. `https://dashboard.polilladigital.cl` o
     `https://tu-usuario.github.io`).
   - **Authorized redirect URIs**: la misma URL exacta (sin slash final).
5. Copia el **Client ID** generado y pégalo en `index.html`, en la constante
   `CID` (cerca de la línea 659):
   ```js
   const CID='TU_GOOGLE_OAUTH_CLIENT_ID.apps.googleusercontent.com';
   ```
6. Ajusta también la constante `REDIR` justo debajo, con la misma URL que
   pusiste como redirect URI:
   ```js
   const REDIR='https://dashboard.polilladigital.cl';
   ```

### 2. Cloudflare Worker — proxy de Anthropic y Meta

El Worker guarda tus API keys secretas y responde solo a peticiones del
dashboard.

1. Crea una cuenta en [Cloudflare](https://dash.cloudflare.com/) si no tienes.
2. Instala Wrangler (CLI de Cloudflare) y autentícate:
   ```bash
   npm install -g wrangler
   wrangler login
   ```
3. Despliega `cloudflare-worker.js` como un Worker nuevo:
   ```bash
   wrangler deploy cloudflare-worker.js --name polilla-digital-proxy
   ```
   (o pega el contenido del archivo directo en el editor del dashboard de
   Cloudflare, en **Workers & Pages → Create → Create Worker**).
4. Configura los secrets del Worker (**Settings → Variables → Secrets**, o
   por CLI):
   ```bash
   wrangler secret put ANTHROPIC_API_KEY
   wrangler secret put META_TOKEN
   ```
   - `ANTHROPIC_API_KEY`: tu API key de [console.anthropic.com](https://console.anthropic.com/).
   - `META_TOKEN`: un token de acceso de larga duración de un System User en
     tu [Meta Business Manager](https://business.facebook.com/) con permisos
     de lectura sobre las cuentas publicitarias que quieras ver en el
     dashboard.
5. En `cloudflare-worker.js`, ajusta `ALLOWED_ORIGIN` con la URL donde
   publiques el dashboard (debe coincidir exactamente, sin slash final):
   ```js
   const ALLOWED_ORIGIN = 'https://dashboard.polilladigital.cl';
   ```
6. Copia la URL pública que te da el Worker (algo como
   `https://polilla-digital-proxy.tu-subdominio.workers.dev`) y pégala en
   `index.html`, en la constante `WORKER_URL` (cerca de la línea 662):
   ```js
   const WORKER_URL='https://TU-WORKER.TU-SUBDOMINIO.workers.dev';
   ```

> Nota: si algún cliente necesita un token de Meta distinto al de la cuenta
> por defecto, se agrega como secret adicional `META_TOKEN_<CLAVE>` en el
> Worker y esa `<CLAVE>` se ingresa en el campo "Meta Token Key" del cliente
> dentro del dashboard.

### 3. Publicar y apuntar el dominio propio

1. En GitHub, ve a **Settings → Pages** de este repo y activa GitHub Pages
   sobre la rama `main` (carpeta raíz).
2. Si quieres usar un dominio propio (ej. `dashboard.polilladigital.cl`):
   - En **Settings → Pages → Custom domain**, escribe el subdominio y guarda.
   - En el proveedor de DNS de `polilladigital.cl`, crea un registro
     **CNAME** para `dashboard` que apunte a `tu-usuario.github.io`.
   - Espera la propagación DNS y activa **Enforce HTTPS** en Pages una vez
     que el certificado esté listo.
3. Verifica que la URL final coincida exactamente (mismo protocolo, sin
   slash final) con `REDIR` en `index.html` y con `ALLOWED_ORIGIN` en el
   Worker — si no coinciden, el login de Google o las llamadas al Worker
   fallarán por CORS/redirect_uri_mismatch.

### 4. Logo propio

El sidebar tiene un botón para subir un logo (clic sobre el logo actual). Si
prefieres dejarlo fijo en el código, sube tu isotipo a este repo (ej.
`logo.png` o `logo.svg`) y referencia esa ruta en el `src` del `<img
id="logo-img">` en `index.html`.

## Desarrollo

No hay build ni dependencias — es HTML/CSS/JS plano. Para probar cambios
localmente basta abrir `index.html` en el navegador o servirlo con cualquier
servidor estático:

```bash
python3 -m http.server 8000
```

Después de editar el `<script>` de `index.html`, valida que el JS siga
siendo sintácticamente correcto con:

```bash
node -e "const fs=require('fs');const html=fs.readFileSync('index.html','utf8');const m=html.match(/<script>([\s\S]*?)<\/script>/);new Function(m[1]);console.log('OK')"
```
