# CLA Bot Setup Guide — LOUST Open Source Projects

> Guía paso a paso para configurar el bot de CLA (Contributor License Agreement)
> en todos los repositorios open source de **@louzt** (personal) y **LOUST-PRO** (org).

## Resumen

El CLA se firma **automáticamente via GitHub**. Cuando alguien abre un PR por primera vez:

1. El bot de CLA detecta que no ha firmado
2. Comenta en el PR con instrucciones
3. El contribuidor comenta: `I have read the CLA Document and I hereby sign the CLA`
4. El bot registra la firma y desbloquea el PR
5. Todos los futuros PRs de ese usuario quedan cubiertos

**Herramienta**: [contributor-assistant/github-action](https://github.com/contributor-assistant/github-action) v2.6.1  
**Almacenamiento de firmas**: [`signatures/version1/cla.json`](signatures/version1/cla.json) en la rama `main` de este repo  
**Documento CLA público**: https://loust.pro/opensource/cla

---

## Repositorios que necesitan el CLA

### @louzt (cuenta personal)
| Repo | URL |
|------|-----|
| SSHboozt DEX Upgrade | `github.com/louzt/SSHboozt_DEXUpgrade` |
| NetBoozt Internet Upgrade | `github.com/louzt/NetBoozt_InternetUpgrade` |

### LOUST-PRO (organización)
| Repo | URL |
|------|-----|
| Cualquier repo público futuro | `github.com/LOUST-PRO/*` |

> **NOTA**: `loust-platform` y `nexus-engine` son **privados** y NO necesitan CLA público.

---

## Paso 1: Crear un Personal Access Token (PAT)

El bot necesita un PAT para poder escribir comentarios y crear la rama de firmas.

1. Ve a → https://github.com/settings/tokens?type=beta (Fine-grained tokens)
2. Click **"Generate new token"**
3. Configura:
   - **Token name**: `CLA Bot - LOUST`
   - **Expiration**: 1 year (máximo) o "No expiration" (classic tokens)
   - **Resource owner**: Selecciona **tu cuenta** (`louzt`)
   - **Repository access**: "Only select repositories" → selecciona:
     - `louzt/cla-signatures`
     - `louzt/SSHboozt_DEXUpgrade`
     - `louzt/NetBoozt_InternetUpgrade`
   - **Permissions**:
     - **Contents**: Read and write (para crear la rama `cla-signatures`)
     - **Pull requests**: Read and write (para comentar en PRs)
     - **Issues**: Read and write (para leer comentarios)
4. Click **"Generate token"**
5. **Copia el token** (solo se muestra una vez)

### Para repos de LOUST-PRO (org):

Si en el futuro agregas repos públicos a la org:
1. Repite el proceso pero selecciona **LOUST-PRO** como Resource Owner
2. O usa un Classic token con scopes: `repo`, `workflow`

---

## Paso 2: Guardar el token como Secret

### Para repos de @louzt:

Tienes dos opciones:

#### Opción A: Secret por repositorio (más seguro)

Para **cada** repo, haz:

```bash
# SSHboozt
gh secret set CLA_TOKEN --repo louzt/SSHboozt_DEXUpgrade

# NetBoozt
gh secret set CLA_TOKEN --repo louzt/NetBoozt_InternetUpgrade
```

Cuando te pida el valor, pega el PAT del Paso 1.

#### Opción B: Via GitHub UI

1. Ve al repo → Settings → Secrets and variables → Actions
2. Click "New repository secret"
3. Name: `CLA_TOKEN`
4. Value: (pega el PAT)
5. Click "Add secret"

### Para repos de LOUST-PRO (org):

Para facilitar mantenimiento, usa un **Organization Secret**:

1. Ve a → https://github.com/organizations/LOUST-PRO/settings/secrets/actions
2. Click "New organization secret"
3. Name: `CLA_TOKEN`
4. Value: (pega el PAT)
5. Repository access: "Selected repositories" → selecciona los repos públicos
6. Click "Add secret"

---

## Paso 3: Copiar el workflow a cada repo

Copia el archivo [`.github/workflows/cla.yml`](.github/workflows/cla.yml) a cada repositorio open source.

### Copia manual (SSH/local):

```bash
# Desde tu máquina local donde están los repos

# SSHboozt
mkdir -p ~/SSHboozt_DEXUpgrade/.github/workflows/
cp cla.yml ~/SSHboozt_DEXUpgrade/.github/workflows/cla.yml

# NetBoozt
mkdir -p ~/NetBoozt_InternetUpgrade/.github/workflows/
cp cla.yml ~/NetBoozt_InternetUpgrade/.github/workflows/cla.yml
```

### Copia via GitHub CLI:

```bash
# Leer el contenido del workflow y crear en cada repo
gh api repos/louzt/SSHboozt_DEXUpgrade/contents/.github/workflows/cla.yml \
  --method PUT \
  --field message="chore: add CLA bot workflow" \
  --field content="$(base64 -w0 cla.yml)" \
  --field branch="main"

gh api repos/louzt/NetBoozt_InternetUpgrade/contents/.github/workflows/cla.yml \
  --method PUT \
  --field message="chore: add CLA bot workflow" \
  --field content="$(base64 -w0 cla.yml)" \
  --field branch="main"
```

### El archivo `cla.yml` (centralizado):

Este repo usa firmas centralizadas. Todos los repos deben apuntar a `louzt/cla-signatures`:

```yaml
name: "📝 CLA Assistant"

on:
  issue_comment:
    types: [created]
  pull_request_target:
    types: [opened, closed, synchronize]

permissions:
  actions: write
  contents: write
  pull-requests: write
  statuses: write

jobs:
  cla:
    name: "CLA Check"
    runs-on: ubuntu-latest
    if: |
      (github.event_name == 'pull_request_target') ||
      (github.event_name == 'issue_comment' && (
        github.event.comment.body == 'recheck' ||
        github.event.comment.body == 'I have read the CLA Document and I hereby sign the CLA'
      ))
    steps:
      - name: "CLA Assistant"
        uses: contributor-assistant/github-action@v2.6.1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PERSONAL_ACCESS_TOKEN: ${{ secrets.CLA_TOKEN }}
        with:
          remote-organization-name: louzt
          remote-repository-name: cla-signatures
          path-to-signatures: "signatures/version1/cla.json"
          path-to-document: "https://loust.pro/opensource/cla"
          branch: "main"
          allowlist: "bot*,dependabot*,renovate*,github-actions*,loust-bot*"

          custom-notsigned-prcomment: |
            ¡Gracias por tu contribución! 🎉

            Antes de que podamos aceptar tu Pull Request, necesitas firmar nuestro
            **Acuerdo de Licencia de Contribuidor (CLA)**.

            📄 **[Lee el CLA completo aquí](https://loust.pro/opensource/cla)**

            ---

            ### ✍️ Para firmar, comenta exactamente:

            ```
            I have read the CLA Document and I hereby sign the CLA
            ```

            Solo necesitas firmarlo **una vez**. Todos tus futuros PRs en cualquier
            repositorio de **@louzt** y **LOUST-PRO** estarán cubiertos.

            ---

            ℹ️ Si contribuyes en nombre de una empresa, asegúrate de tener
            autorización de tu empleador.

          custom-signed-prcomment: |
            ✅ **CLA firmado exitosamente.** Tu Pull Request está listo para revisión.

          custom-allsigned-prcomment: |
            ✅ Todos los contribuidores han firmado el CLA.
```

---

## Paso 4: Verificar que funciona

### Test rápido:

1. Crea una cuenta de prueba en GitHub (o pide a un amigo)
2. Haz fork del repo
3. Crea un cambio mínimo (edita el README o algo)
4. Abre un PR
5. El bot debe comentar automáticamente pidiendo la firma
6. Comenta: `I have read the CLA Document and I hereby sign the CLA`
7. El bot debe marcar el check como passed

### Verificar las firmas:

Después de la primera firma, el bot actualiza [`signatures/version1/cla.json`](signatures/version1/cla.json):

```json
{
  "signedContributors": [
    {
      "name": "username",
      "id": 12345678,
      "comment_id": 9876543,
      "created_at": "2026-03-02T12:00:00Z",
      "repoId": 123456
    }
  ]
}
```

### Verificar el status check:

En la pestaña de Checks del PR, deberías ver:
- ✅ `CLA Check` — Si ya firmó
- ❌ `CLA Check` — Si falta firma (bloquea merge)

---

## Paso 5 (Opcional): Proteger la rama main con CLA required

Para **forzar** que nadie pueda hacer merge sin CLA:

1. Ve al repo → Settings → Branches → Branch protection rules
2. Click "Add rule" (o edita la regla de `main`)
3. Configura:
   - Branch name pattern: `main`
   - ✅ Require status checks to pass before merging
   - Search y agrega: **"CLA Check"**
   - ✅ Require branches to be up to date before merging
4. Click "Save changes"

Ahora es **imposible** hacer merge sin CLA firmado.

---

## Troubleshooting

### El bot no comenta

1. **Verifica que el workflow existe** en `.github/workflows/cla.yml`
2. **Verifica el secret** `CLA_TOKEN` en Settings → Secrets
3. **Revisa Actions**: Ve a la pestaña Actions del repo y busca errores
4. **Permiso del PAT**: Asegúrate de que el PAT tiene permisos sobre ese repo

### El bot comenta pero no registra la firma

1. El comentario debe ser **exacto**: `I have read the CLA Document and I hereby sign the CLA`
2. No puede tener espacios extra, emojis, ni nada adicional
3. Verifica que el PAT (`CLA_TOKEN`) tiene permiso de **write** en Contents del repo `louzt/cla-signatures`

### Error "Resource not accessible by integration"

Esto pasa cuando `GITHUB_TOKEN` no tiene suficientes permisos:
- Verifica que el workflow tiene la sección `permissions` con `contents: write` y `pull-requests: write`
- En Settings → Actions → General → Workflow permissions → selecciona "Read and write permissions"

---

## Resumen de acciones requeridas

| # | Acción | Dónde | Tiempo estimado |
|---|--------|-------|-----------------|
| 1 | Crear PAT con permisos | github.com/settings/tokens | 2 min |
| 2 | Guardar como secret `CLA_TOKEN` | Cada repo (Settings → Secrets) | 1 min/repo |
| 3 | Copiar `cla.yml` al repo | `.github/workflows/cla.yml` | 1 min/repo |
| 4 | (Opcional) Proteger rama `main` | Cada repo (Settings → Branches) | 2 min/repo |
| 5 | Test con un PR de prueba | Fork → PR → Firma | 5 min |

**Total**: ~15 minutos para tener CLA automático en todos los repos.