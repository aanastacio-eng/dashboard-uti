# Panel Uti 1 / Uti 2 — Inventario y Ventas

Dashboard estático (HTML/CSS/JS, sin frameworks) que lee `data/data.json` y se
refresca solo cada 60 s. Ya viene cargado con un snapshot real de tus 74
productos en las colecciones **Uti 1** y **UTI 2** (extraído directo de Odoo).

## 1. Subir esto a GitHub

```bash
git init
git add .
git commit -m "Dashboard Uti 1 / Uti 2"
git remote add origin https://github.com/TU_USUARIO/TU_REP.git
git branch -M main
git push -u origin main
```

## 2. Publicarlo (y sobre "privado")

⚠️ **Importante:** GitHub Pages normal es público — cualquiera con el link
puede verlo, aunque el repositorio sea privado (a menos que tengas **GitHub
Enterprise**, que sí soporta Pages verdaderamente privado). Para tu caso
("privado / interno") tienes 3 opciones reales, de más simple a más robusta:

| Opción | Cómo | Nivel de privacidad |
|---|---|---|
| A. GitHub Pages + Cloudflare Access | Pages normal detrás de un proxy de Cloudflare que pide login (gratis) | Alto — login por correo/SSO |
| B. Netlify / Vercel con "Password Protection" | Deploy del mismo `index.html`, activas contraseña en el panel | Alto — 1 contraseña compartida |
| C. GitHub Pages normal | Solo si el link no se comparte públicamente | Bajo — "seguridad por oscuridad" |

Puedo dejarte armada la opción A o B si me confirmas cuál prefieres — ambas
reusan exactamente estos mismos archivos.

## 3. El pipeline en tiempo real: Odoo → n8n → GitHub

Así es como cada venta actualiza el dashboard sin que nadie toque nada:

**Paso 1 — Odoo dispara el evento**
En Odoo, crea una *Automatización* (Ajustes → Técnico → Automatizaciones):
- Modelo: `Sale Order` (o `Stock Move` si prefieres disparar en la salida de
  bodega, no en la confirmación de la venta)
- Disparador: "Al confirmar" (on create/write, `state = 'sale'`)
- Acción: **Webhook** apuntando a tu workflow de n8n
  (`https://TU-N8N/webhook/uti-ventas`)

**Paso 2 — n8n recalcula y arma el JSON**
Workflow con 3 pasos:
1. **Webhook** (nodo trigger) — recibe el aviso de Odoo.
2. **Odoo node / HTTP Request** — vuelve a consultar `product.template`
   filtrando `x_studio_coleccin_1 in [214, 218]` (Uti 1 / UTI 2) con los
   campos: `qty_available`, `x_studio_stock_inicial`, `sales_count`,
   `categ_id`, `x_studio_coleccin_1`. Transforma el resultado exactamente al
   esquema de `data/data.json` (ver más abajo).
3. **HTTP Request → GitHub Contents API** — sube el JSON nuevo al repo:

   ```
   PUT https://api.github.com/repos/TU_USUARIO/TU_REPO/contents/data/data.json
   Headers: Authorization: Bearer TU_GITHUB_TOKEN
   Body:
   {
     "message": "Actualización automática de ventas",
     "content": "<JSON en base64>",
     "sha": "<sha del archivo actual, se obtiene con GET al mismo endpoint>",
     "branch": "main"
   }
   ```

   El token de GitHub (Personal Access Token con permiso `contents: write`
   sobre ese repo) se guarda como credencial dentro de n8n — nunca en el
   dashboard ni en el repo.

**Paso 3 — El dashboard se entera solo**
`index.html` hace `fetch('data/data.json')` cada 60 s. En cuanto GitHub
Pages sirve el commit nuevo, los números, gráficos y tabla se actualizan sin
recargar la página.

> Si quieres, en la próxima conversación puedo construirte ese workflow de
> n8n directamente (ya tengo acceso a tus herramientas de n8n) — solo
> necesito que me confirmes el nombre del repo de GitHub y que tengas
> guardado ahí un token con permiso de escritura.

## 4. Esquema exacto de `data/data.json`

```json
{
  "actualizado": "2026-08-13T00:00:00Z",
  "kpis": {
    "inventario_inicial": 788,
    "inventario_actual": -286,
    "ventas_totales": 6367,
    "ingresos_totales": 123456.78,
    "total_productos": 74
  },
  "por_categoria": { "Camisas y tops": {"ventas": 0, "ingresos": 0, "productos": 0}, "...": {} },
  "top_vendidos": [ {"nombre": "...", "coleccion": "Uti 1", "categoria": "...", "ventas": 0, "stock_actual": 0} ],
  "menos_vendidos": [ "... mismo formato ..." ],
  "ventas_por_talla": { "XS": 0, "S": 0, "M": 0, "L": 0, "XL": 0 },
  "productos": [ {"odoo_id":0,"nombre":"","coleccion":"","categoria":"","stock_inicial":0,"stock_actual":0,"ventas":0,"precio":0} ]
}
```

⚠️ **Nota sobre "ventas por talla":** el snapshot actual usa una
distribución estimada (no viene de Odoo todavía), porque requiere agregar
las ventas a nivel de *variante* (no de plantilla de producto) usando el
atributo `Size`. Cuando construya el workflow de n8n puedo calcularlo real
sin problema — solo toma una consulta adicional a `product.product`.

## 5. Archivos de este repo

- `index.html` — el dashboard (todo en un archivo).
- `data/data.json` — snapshot inicial real de Uti 1 + UTI 2 (74 productos).
- `build_data.py` — script usado para generar ese snapshot desde Odoo (útil
  como referencia si prefieres correr el refresco por cron en vez de n8n).
