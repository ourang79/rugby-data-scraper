# Rugby CyL — Datos iSquad

Scraper automático que descarga datos de [resultadosrugby.isquad.es](https://resultadosrugby.isquad.es) 
para la Federación Territorial de Castilla y León de Rugby.

## Cómo funciona

1. GitHub Actions ejecuta el scraper cada 3 horas
2. Descarga clasificaciones, resultados y estadísticas de cada torneo
3. Guarda los datos como JSON en la carpeta `data/`
4. El plugin de WordPress lee estos JSON directamente desde GitHub

## Archivos de datos

- `data/index.json` — Índice con todos los torneos y última actualización
- `data/521.json` — Liga Senior Femenino
- `data/324.json` — Liga Senior Masculina
- etc.

## Configurar torneos

Editar el bloque `TORNEOS` en `.github/workflows/scrape.yml`:

```yaml
TORNEOS: |
  [
    {"id": "521", "nombre": "Liga Senior Femenino"},
    {"id": "324", "nombre": "Liga Senior Masculina"}
  ]
```

## Uso desde WordPress

El plugin iSquad-SportsPress-Sync lee los datos desde:

```
https://raw.githubusercontent.com/TU_USUARIO/rugby-data-scraper/main/data/521.json
```

## Ejecución manual

```bash
# Local
python scripts/scrape.py

# O desde GitHub: Actions → Scrape iSquad Rugby Data → Run workflow
```
