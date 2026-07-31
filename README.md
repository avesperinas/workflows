# workflows

Reusable workflows de GitHub Actions para el homelab.

No contiene secretos. Es público a propósito, para que cualquier repo de app pueda
referenciarlo sin configurar permisos entre repos.

## `release.yml`

Construye la imagen de una app, la publica en `ghcr.io/avesperinas/<repo>` y escribe
la nueva versión en [`avesperinas/infra`](https://github.com/avesperinas/infra), que
es la fuente de verdad de lo que está desplegado.

Se dispara desde el repo de la app al hacer push de un tag `v*`:

```yaml
name: release
on: { push: { tags: ['v*'] } }
jobs:
  release:
    uses: avesperinas/workflows/.github/workflows/release.yml@main
    with:
      app_name: caudal
    secrets:
      infra_token: ${{ secrets.INFRA_TOKEN }}
```

### Inputs

| Input | Req. | Default | Descripción |
|---|---|---|---|
| `app_name` | sí | — | Carpeta del stack: `infra/stacks/<app_name>` |
| `dockerfile` | no | `Dockerfile` | Ruta al Dockerfile |
| `context` | no | `.` | Contexto de build |
| `image_name` | no | nombre del repo | Nombre bajo `ghcr.io/avesperinas/` |
| `bump_infra` | no | `true` | Si `false`, solo construye: no commitea en `infra` |
| `infra_repo` | no | `avesperinas/infra` | Repo de estado desplegado |

### Repos con más de una imagen

`image_name` + `bump_infra` permiten publicar varias imágenes desde el mismo tag.
Solo una de las llamadas debe escribir en `infra`:

```yaml
jobs:
  app:
    uses: avesperinas/workflows/.github/workflows/release.yml@main
    with: { app_name: caudal }
    secrets: { infra_token: ${{ secrets.INFRA_TOKEN }} }

  backup:
    uses: avesperinas/workflows/.github/workflows/release.yml@main
    with:
      app_name: caudal
      image_name: caudal-backup
      context: scripts/backup
      dockerfile: scripts/backup/Dockerfile
      bump_infra: false          # el job `app` ya publica la versión
    secrets: { infra_token: ${{ secrets.INFRA_TOKEN }} }
```

Ambas imágenes comparten el mismo `IMAGE_TAG`, así que el stack las despliega juntas.

### Secrets

| Secret | Descripción |
|---|---|
| `infra_token` | PAT fine-grained, `Contents: read/write` **solo** sobre `infra` |

### Qué hace

1. **`build`** — valida que el tag sea `vX.Y.Z` (rechaza `latest`), construye con
   buildx y cache de Actions, y publica dos tags inmutables:
   `:vX.Y.Z` y `:sha-<7 chars>`.
2. **`bump`** — escribe `stacks/<app_name>/versions.env` con `IMAGE_TAG=vX.Y.Z` y
   commitea en `infra`. Si el stack no existe, falla con un mensaje claro: no lo
   crea solo, porque un stack necesita compose, healthcheck e ingress pensados.

Ante dos apps publicando a la vez, el push se reintenta hasta 5 veces reescribiendo
el fichero sobre la punta actual de `main`, así que nunca hay conflicto de merge.

El despliegue en sí no ocurre aquí: el server reconcilia `infra` por su cuenta cada
60 s. Ver el README de `infra`.
