# Deployment Guide — pos-mobile-packages

Central guide for publishing and deploying artifacts to GitHub Packages registry.

## 🏗️ Arquitetura de Deploy

`pos-mobile-packages` é um **registry Maven central** que publica múltiplos artefatos POS:

### Por que publicar aqui?

- GitHub Packages **não aceita GitHub App tokens** para `packages:write`
- Só aceita **PAT clássico** ou **`GITHUB_TOKEN` automático**
- O `GITHUB_TOKEN` nativo só funciona para pacotes do **mesmo repo**
- Pacotes `co.stone.pos.mobile:*` pertencem a este repo → publicam aqui

### Padrão de Deploy Cross-Repo

Quando um pacote é desenvolvido em outro repo:

1. **Source Repository (ex: pos-android-hal):** Cria release com tag
2. **pos-mobile-packages:** Checkout cross-repo e publica no registry

---

## 📦 Publicar HAL API em GitHub Packages

### Contexto

O `hal-api` é desenvolvido em `pos-android-hal`, mas publicado aqui porque segue o padrão acima.

### Fluxo de Release

1. **pos-android-hal:** Cria tag `hal-api/x.y.z` + GitHub Release (workflow "HAL API Release")
2. **pos-mobile-packages:** Publica AAR em GitHub Packages (workflow "Publish HAL API to GitHub Packages")

### Como Publicar

#### Pré-requisitos

- [ ] Release foi criada em `pos-android-hal` com tag `hal-api/x.y.z`
- [ ] GitHub Packages registry em `pos-mobile-packages` está acessível
- [ ] Secrets org-level estão configurados:
  - `GH_HARDWARE_MAMBA_SETUP_PRIVATE_KEY` (GitHub App private key para cross-repo checkout)
  - `PACKAGECLOUD_READ_TOKEN` (PackageCloud token para dependencies)
  - `PACKAGECLOUD_READ_TOKEN_INTERNAL` (PackageCloud internal token)

#### Steps

1. **Acesse o repositório `pos-mobile-packages` no GitHub**

2. **Abra a aba "Actions"**

3. **Selecione o workflow "Publish HAL API to GitHub Packages"**

4. **Clique em "Run workflow"** (botão azul)

5. **Preencha o campo "Tag do hal-api"** com o formato `hal-api/x.y.z`
   - Exemplo: `hal-api/3.6.6`
   - Deve corresponder à tag criada em `pos-android-hal`

6. **Clique em "Run workflow"** para executar

7. **Monitore o workflow** na aba Actions até que complete (⏳ ~5-10 minutos)

#### Resultado Esperado

Se bem-sucedido:
- AAR será publicado em `https://maven.pkg.github.com/stone-payments/pos-mobile-packages`
- Será acessível para projetos que dependem do `hal-api`

#### Troubleshooting

| Erro | Causa Possível | Solução |
|------|---|---|
| `Tag inválida: 'xxx'. Formato esperado: hal-api/x.y.z` | Formato de tag incorreto | Verifique a tag em pos-android-hal (deve ser `hal-api/3.6.6`, não `3.6.6` ou `hal-api-3.6.6`) |
| `fatal: reference is not a tree: hal-api/x.y.z` | Tag não existe em pos-android-hal | Confirme que o release foi criado em pos-android-hal e aguarde alguns segundos |
| `401 Unauthorized` ou erro de autenticação | GitHub App token ou GITHUB_TOKEN inválido | Verifique que os secrets estão configurados corretamente em org-level |
| `Build failed` | Dependências faltando ou build quebrado | Verifique que `PACKAGECLOUD_READ_TOKEN*` estão corretos |

### Secrets Necessários

Todos os secrets abaixo devem estar **em nível de organização** (`stone-payments`) e acessíveis a `pos-mobile-packages`:

```yaml
GH_HARDWARE_MAMBA_SETUP_PRIVATE_KEY    # GitHub App private key (para checkout pos-android-hal)
PACKAGECLOUD_READ_TOKEN                # Token de leitura PackageCloud
PACKAGECLOUD_READ_TOKEN_INTERNAL       # Token de leitura PackageCloud (internal)
GITHUB_TOKEN                           # Gerado automaticamente pelo GitHub (sempre disponível)
```

### Workflow File

O workflow está definido em `.github/workflows/publish-hal-api.yml`:

```yaml
on:
  workflow_dispatch:
    inputs:
      tag:
        description: 'Tag do hal-api criada em pos-android-hal (ex: hal-api/3.6.6)'
        required: true
```

**Permissões:**
- `contents: read` — checkout do código
- `packages: write` — publicar em GitHub Packages

### Referências

- Workflow: `.github/workflows/publish-hal-api.yml`
- Registry: `https://maven.pkg.github.com/stone-payments/pos-mobile-packages`
- Documentação: [GitHub Packages Maven Guide](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-maven-registry)

---

## 🚀 Adicionar Novos Deployments

Para publicar um novo pacote `co.stone.pos.mobile:*`:

### 1. Criar Workflow

Crie um novo workflow em `.github/workflows/publish-<package>.yml`:

```yaml
name: Publish <Package Name> to GitHub Packages

on:
  workflow_dispatch:
    inputs:
      tag:
        description: 'Tag da release (ex: <package>/1.0.0)'
        required: true

permissions:
  contents: read
  packages: write

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source repo
        uses: actions/checkout@v4
        with:
          repository: stone-payments/<source-repo>
          ref: ${{ inputs.tag }}
          token: ${{ secrets.GH_HARDWARE_MAMBA_SETUP_PRIVATE_KEY || secrets.GITHUB_TOKEN }}
      
      - name: Publish to GitHub Packages
        run: ./gradlew :<package>:publishAllPublicationsToGitHubPackagesRepository
        env:
          GITHUB_PUBLISH_USERNAME: x-access-token
          GITHUB_PUBLISH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GH_PACKAGES_REPOSITORY_URL: https://maven.pkg.github.com/stone-payments/pos-mobile-packages
```

### 2. Configurar Secrets

Adicione ao nível de organização em `stone-payments`:
- `GH_HARDWARE_MAMBA_SETUP_PRIVATE_KEY` (se cross-repo)
- `PACKAGECLOUD_READ_TOKEN*` (se aplicável)

### 3. Documentar

Adicione uma seção similar a "Publicar HAL API" neste guia.

### 4. Testar

Rode o workflow com uma tag de teste e valide:
- ✅ Workflow completa sem erros
- ✅ Pacote aparece em `https://maven.pkg.github.com/stone-payments/pos-mobile-packages`
- ✅ Consumidor consegue fazer download

---

## 📝 Deployments Ativos

| Pacote | Fonte | Workflow | Status |
|--------|-------|----------|--------|
| `hal-api` | `pos-android-hal` | `publish-hal-api.yml` | ✅ Ativo |

---

**Última atualização:** 2026-08-19
