# Deploy — UFI Barcelona

Despliegue en producción. **Cero comandos locales**: ambos servicios (frontend + backend) auto-despliegan desde GitHub en cada push a `main`.

---

## Arquitectura de deploy

```
                                ┌──────────────┐
git push origin main  ────►    │   GitHub     │
                                │  bcn_traffic │
                                └──┬────────┬──┘
                          webhook  │        │  webhook
                                   ▼        ▼
                          ┌─────────┐    ┌─────────┐
                          │ Vercel  │    │ Railway │
                          │ frontend│    │ backend │
                          └─────────┘    └─────────┘
                                │             │
                                ▼             ▼
                       ufi-bcn.vercel.app   xxx.up.railway.app
```

---

## Configuración Vercel (frontend)

### Setup inicial (ya hecho)

| Setting | Valor |
|---|---|
| Repo | `EpicGatuman/bcn_traffic` |
| Production Branch | `main` |
| Root Directory | `frontend` |
| Framework Preset | Vite |
| Build Command | `npm run build` (en `frontend/vercel.json`) |
| Output Directory | `dist` |
| Custom Domain | `ufi-bcn.vercel.app` |

### Variables de entorno (Settings → Environment Variables)

| Key | Value |
|---|---|
| `VITE_API_BASE_URL` | URL pública de Railway (ej: `https://xxx.up.railway.app`) |

### Deployment Protection

Settings → Deployment Protection → **Disabled**. Sin esto, visitantes anónimos reciben 403.

---

## Configuración Railway (backend)

### Setup inicial

1. Railway dashboard → **New Project** → **Deploy from GitHub repo** → `EpicGatuman/bcn_traffic`
2. Railway detecta `Dockerfile` raíz + `devops/infra/railway.json`
3. Settings → Networking → **Generate Domain** → da `xxx.up.railway.app`

### Variables de entorno (Variables tab)

| Key | Value |
|---|---|
| `ANTHROPIC_API_KEY` | `sk-ant-...` (clave Anthropic) |
| `CORS_ORIGINS` | `https://ufi-bcn.vercel.app` |

### Healthcheck

Railway pega a `/health` automático (definido en `devops/infra/railway.json`). Si responde 200, deploy = Ready.

Verificación manual: `https://xxx.up.railway.app/health` debe devolver JSON.

---

## Flujo normal de deploy

1. Edita código en local
2. `git commit`
3. `git push origin main`
4. Vercel + Railway reciben webhook → builds en paralelo (~1-3 min cada uno)
5. Listo

---

## Regenerar Parquet UFI (datos)

El backend sirve `data/processed/ufi_latest.parquet`. Está committeado al repo — Railway lo copia en la imagen Docker en cada build.

Si cambias modelos o quieres datos frescos:

```powershell
cd data-ml
python -m ml.score
cd ..
git add data/processed/ufi_latest.parquet `
        data/processed/barrios.geojson `
        data/processed/tramos.geojson `
        data/processed/mapping_barrios.csv
git commit -m "data: actualizar UFI"
git push origin main
```

Railway rebuild incluye los archivos nuevos en la imagen.

---

## Troubleshooting

### Vercel: "Root Directory does not exist"
Settings → Build and Deployment → Root Directory = `frontend` (sin `./`, sin espacios).

### Vercel: 403 Forbidden al abrir la URL
Settings → Deployment Protection → Disabled.

### Vercel: front carga pero `/api/...` 404
Falta `VITE_API_BASE_URL` o tiene URL incorrecta. Vuelve a Environment Variables, corrige, Redeploy con cache OFF.

### Railway: build falla con "no Dockerfile"
`devops/infra/railway.json` debe apuntar a `Dockerfile` raíz. Comprueba `dockerfilePath`.

### Railway: backend responde pero front da CORS error
Variable `CORS_ORIGINS` en Railway no incluye la URL de Vercel. Añade `https://ufi-bcn.vercel.app` y redeploy.

### Railway: `/explain` devuelve plantilla en vez de Claude
Falta `ANTHROPIC_API_KEY` o está mal escrita. Comprueba en Railway Variables.

---

## URLs de referencia

- Frontend: https://ufi-bcn.vercel.app
- Backend: `https://<railway-domain>.up.railway.app` (rellena tras setup)
- Repo: https://github.com/EpicGatuman/bcn_traffic
- Docs API: `<railway-domain>/docs`
