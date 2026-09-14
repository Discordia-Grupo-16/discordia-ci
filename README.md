# discordia-ci

Workflows reutilizables de CI para los repos de Discordia. No tiene código de producto:
solo `.github/workflows/`.

Cada repo de servicio tiene un `ci.yml` de ~20 líneas que apunta acá. Si hay que arreglar
algo del pipeline, se toca un archivo y se mueve el tag `v1`, en vez de abrir un PR por
repo.

## Workflows disponibles

| Archivo | Repos |
| --- | --- |
| `python-service.yml` | `identity`, `community`, `mod`, `monetization` |
| `go-service.yml` | `api-gateway`, `chat-and-real-time`, `notifications`, `metrics` — **pendiente** |
| `node-app.yml` | `web-app`, `backoffice`, `mobile` — **pendiente** |

Los dos pendientes se escriben cuando esos repos tengan código. Un workflow contra un
repo vacío es adivinar el layout, y eso ya nos costó media tarde en `community`.

## Cómo se usa

En el repo del servicio, `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [develop, main]
  pull_request:
    branches: [develop, main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    uses: Discordia-Grupo-16/discordia-ci/.github/workflows/python-service.yml@v1
    with:
      test-db-name: community_test_db
      coverage-threshold: 70
      extra-env: |
        IDENTITY_SERVICE_URL=http://localhost:8001
        BUS_URL=redis://localhost:6379
```

`test-db-name` es obligatorio y tiene que coincidir con lo que espera el `conftest.py` del
repo. Si no coincide, todos los tests fallan con `database does not exist`.

## Versionado

Los repos apuntan a `@v1`, que es un tag **movible**:

```bash
git tag -f v1 && git push -f origin v1
```

Se corrige un bug del pipeline y todos los repos lo toman en el próximo push.

## Contrato de repositorio

Los repos tienen que cumplir el layout definido en `ci/contrato-ci.md` del repo de
documentación: dónde va cada archivo, qué trae cada repo desde el primer commit, y qué se
mide para cobertura. Si un repo no lo cumple, su pipeline falla.

## Decisiones que vale conocer

- **gitleaks va con el binario, no con `gitleaks/gitleaks-action@v2`.** La action pide
  `GITLEAKS_LICENSE` en repos de organización y falla en ~5 segundos con un error que no
  lo explica.
- **El job de gitleaks vive dentro de cada workflow por stack**, no en un `security.yml`
  aparte, para que los required status checks se llamen `CI / gitleaks` y no haya que
  reconfigurar branch protection en cada repo.
- **El CI no corre migraciones.** Los tests crean y borran sus tablas.
- **No hay service container del bus.** Ningún test publica eventos todavía. Cuando haya
  tests de integración con el broker, se agrega un input `needs-<broker>` y el bloque de
  service — y recién cuando ADR-0003 esté cerrado.
- **`ruff format --check` está activo**, además de `ruff check`.
- **CodeQL queda para CP2**, y solo en los servicios que tocan auth y permisos. El
  enunciado lo recomienda, no lo exige, y corriéndolo por push en todos los repos come
  minutos de Actions sin aportar nada en repos casi vacíos.
- **Este repo no tiene CI propio.** No hay nada que testear, y la consigna marca el CI
  roto como red line: un pipeline que no aporta es solo superficie de falla.

## Estado de la protección de ramas

Las reglas están configuradas en cada repo pero **GitHub no las aplica en repos privados
con plan Free de organización**. El pipeline corre y el gate falla cuando corresponde,
pero el merge no queda bloqueado técnicamente. Se activa al pasar los repos a públicos o
al plan Team.
