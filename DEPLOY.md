# Deployment Guide — pos-mobile-packages

Este guia explica como fazer deploy e publicar artefatos do projeto.

## 📦 Publicar HAL API em GitHub Packages

### Contexto

O `hal-api` é desenvolvido em `pos-android-hal`, mas publicado em `pos-mobile-packages` GitHub Packages registry porque:

- GitHub Packages **não aceita tokens de GitHub App** como autenticação para `packages:write`
- Só aceita **PAT clássico** ou **`GITHUB_TOKEN` automático**
- O `GITHUB_TOKEN` nativo só funciona para pacotes que pertencem ao **mesmo repositório**
- Como `co.stone.pos.mobile:hal-api-*` é publicado aqui (pos-mobile-packages), o publish deve rodar neste repo

### Fluxo de Release

O release é em **2 passos**:

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

## 📝 Outras Operações

*(Espaço reservado para futuras operações de deploy)*

---

**Última atualização:** 2026-08-19
