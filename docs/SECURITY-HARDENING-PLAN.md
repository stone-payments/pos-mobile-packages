# Security Hardening Plan - publish-hal-api Workflow
## `pos-mobile-packages` PR #2

**Status:** 📋 Design Phase (awaiting review & approval)  
**Date:** 2026-08-06  
**Owner:** eduardo-nunes-stone  
**Scope:** 3 security improvements for POS/Payment supply-chain integrity  
**PR Reference:** https://github.com/stone-payments/pos-mobile-packages/pull/2

---

## Executive Summary

O workflow `publish-hal-api.yml` atual está **seguro**, mas **não fornece prova criptográfica** de que o `.aar` publicado foi realmente construído pelo CI e não foi tampered. Para aplicações de pagamento (POS), isso é crítico.

### Melhorias Propostas

1. **SLSA L3 Attestation** — prova criptográfica do build
2. **Tag Signature Verification** — valida que a tag veio de fonte confiável
3. **Build Artifact SBOM** — lista completa de dependências (compliance)

### Impacto

- ✅ Cumpre requisitos de supply-chain security para POS/Pagamento
- ✅ Detecta builds comprometidos
- ✅ Auditoria completa do que foi publicado

---

## 1. SLSA L3 Attestation

### O que é?

SLSA (Supply-chain Levels for Software Artifacts) é um padrão de segurança que prova:
- ✅ Quem fez o build (GitHub Actions)
- ✅ Quando foi feito (timestamp)
- ✅ De qual código foi gerado (commit hash da tag)
- ✅ Com quais dependências (build environment)

**SLSA L3** = proveniência criptográfica assinada pelo GitHub (equivalente a certificado digital para artefatos).

### Por que Importante para POS?

Cenários de ataque prevenidos:
- ❌ Alguém substitui o `.aar` publicado no GitHub Packages
- ❌ Alguém injeta código malicioso no build
- ❌ Alguém cria um `.aar` falsificado

Com SLSA L3:
- ✅ GitHub assina criptograficamente a proveniência
- ✅ Consumidores podem verificar a assinatura
- ✅ Qualquer tampering é detectado imediatamente

### Pré-requisitos

- ✅ **Já temos:** GitHub Actions (nativo)
- ✅ **Já temos:** Actions OIDC trust (GitHub fornece automaticamente)
- ⏳ **Adicionar:** `id-token: write` nas permissions

### Mudanças no Workflow

#### Permissions

```yaml
# ANTES
permissions:
  contents: read
  packages: write

# DEPOIS
permissions:
  contents: read
  packages: write
  id-token: write        # ← NOVO: para gerar token OIDC
  attestations: write    # ← NOVO: para publicar attestation
```

#### Steps Adicionais

```yaml
- name: Build Release
  id: build
  env:
    PACKAGECLOUD_READ_TOKEN: ${{ secrets.PACKAGECLOUD_READ_TOKEN }}
    PACKAGECLOUD_READ_TOKEN_INTERNAL: ${{ secrets.PACKAGECLOUD_READ_TOKEN_INTERNAL }}
  run: |
    ./gradlew :hal-api:assembleRelease
    
    # Extrair digest SHA256 do AAR gerado
    echo "digest=$(find build -name '*.aar' -exec sha256sum {} \; | head -1 | cut -d' ' -f1)" >> $GITHUB_OUTPUT

- name: Generate SLSA L3 Provenance Attestation
  uses: actions/attest-build-provenance@v1
  with:
    subject-path: |
      build/outputs/release/hal-api-*.aar
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

### O que Acontece Após?

1. GitHub gera attestation JSON assinado com key privada do GitHub
2. Attestation é publicado no GitHub Artifact Attestation registry
3. Consumidores podem verificar:
   ```bash
   gh attestation verify <aar-path> --owner stone-payments
   ```

### No Portal Karavela

Na aba **Dependencies > Build Integrity** do Portal, aparecerá:
```
✅ Build Integrity: VERIFIED (SLSA L3)
   - Builder: github-actions
   - Timestamp: 2026-08-06T18:00:00Z
   - Source Commit: abc1234def5678
```

---

## 2. Tag Signature Verification

### O que é?

Validar que a tag `hal-api/x.y.z` foi criada por um committer autorizado (não um bot malicioso).

### Por que Importante?

Cenário de ataque prevenido:
- ❌ Alguém cria uma tag fake em pos-android-hal apontando para código malicioso
- ❌ Este workflow publica o código comprometido
- ❌ Consumidores baixam código malicioso sem saber

Com verificação de tag signature:
- ✅ Só tags assinadas por keys autorizadas são aceitas
- ✅ Rejeita tags criadas via web UI ou por automações não confiáveis

### Pré-requisitos

- ⏳ **Decisão time:** Obrigatório ou warning?
- ⏳ **Configurar git signing policy** em pos-android-hal:
  - Configurar `require_signed_commits` nas branch protections
  - Distribuir keys GPG/SSH assinadas para team
- ✅ **Já temos:** GitHub Actions consegue verificar tags assinadas

### Steps Adicionais

```yaml
- name: Verify tag signature
  env:
    TAG: ${{ inputs.tag }}
  run: |
    cd pos-android-hal
    
    # Verificar se a tag existe e é válida
    if ! git rev-list -n 1 "$TAG" > /dev/null 2>&1; then
      echo "::error::Tag '$TAG' não existe em pos-android-hal"
      exit 1
    fi
    
    # Verificar assinatura da tag (requer GPG key do author)
    if ! git verify-tag "$TAG" 2>/dev/null; then
      echo "::warning::Tag '$TAG' não está assinada (recomenda-se assinar releases)"
      # Pode ser warning ou error dependendo da política
    fi
    
    # Extrair autor da tag para auditoria
    TAG_AUTHOR=$(git log -1 --format=%an "$TAG")
    echo "Tag '$TAG' criada por: $TAG_AUTHOR"
```

### Comportamento

| Cenário | Comportamento |
|---|---|
| Tag assinada por key autorizada | ✅ Continua |
| Tag não assinada | ⚠️ Warning (permite, mas alerta) |
| Tag assinada por key desconhecida | ❌ Error (bloqueia) |

---

## 3. Build Artifact SBOM (Software Bill of Materials)

### O que é?

SBOM é uma lista detalhada de **todas as dependências** (diretas + transientes) incluídas no `.aar`:

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.4",
  "version": 1,
  "components": [
    {
      "name": "kotlinx-coroutines",
      "version": "1.7.1",
      "type": "library",
      "purl": "pkg:maven/org.jetbrains.kotlinx/kotlinx-coroutines-core@1.7.1"
    }
  ]
}
```

### Por que Importante para POS?

Requisitos de compliance (PCI-DSS, regulações):
- ✅ Rastreabilidade completa do que foi publicado
- ✅ Detecção de dependências vulneráveis (CVEs)
- ✅ Auditoria de licenças (LGPL, GPL, etc.)
- ✅ Prova de origem de cada componente

**Cenário real:**
Uma biblioteca transiente tem CVE descoberto. Com SBOM, você sabe imediatamente quais versões do hal-api são afetadas. Sem SBOM, investigação manual cara.

### Pré-requisitos

- ⏳ Instalar plugin Gradle para gerar SBOM
  - Opção 1: `org.cyclonedx:cyclonedx-gradle-plugin` (recomendado)
  - Opção 2: `syft` + action (manual)

### Mudanças em `build.gradle` (pos-android-hal)

```gradle
plugins {
    // ... existing plugins
    id "org.cyclonedx.bom" version "1.7.0"  // ← NOVO
}

cyclonedx {
    schemaVersion = "1.4"
    outputFormat = "json"
    includeConfigs = ["runtimeClasspath"]
}
```

### Steps no Workflow

```yaml
- name: Build Release with SBOM
  id: build
  env:
    PACKAGECLOUD_READ_TOKEN: ${{ secrets.PACKAGECLOUD_READ_TOKEN }}
    PACKAGECLOUD_READ_TOKEN_INTERNAL: ${{ secrets.PACKAGECLOUD_READ_TOKEN_INTERNAL }}
  run: |
    ./gradlew :hal-api:assembleRelease cycloneDx
    
    # SBOM gerado em: pos-android-hal/build/bom-hal-api-android.json
    echo "sbom-path=pos-android-hal/build/bom-hal-api-android.json" >> $GITHUB_OUTPUT

- name: Upload SBOM as release artifact
  uses: actions/upload-artifact@v4
  with:
    name: sbom
    path: ${{ steps.build.outputs.sbom-path }}
    retention-days: 30

- name: Publish SBOM to GitHub Release
  run: |
    gh release upload ${{ inputs.tag }} ${{ steps.build.outputs.sbom-path }} \
      --repo stone-payments/pos-android-hal
```

---

## Implementation Roadmap

### Phase 1: Design & Review (AGORA)
- [ ] Review deste documento
- [ ] Feedback do team (segurança, SRE)
- [ ] Decisões sobre GPG signing policy (tag signatures)

### Phase 2: SLSA L3 Implementation (1-2 dias)
- [ ] Adicionar `id-token: write` + `attestations: write` ao workflow
- [ ] Implementar step `attest-build-provenance`
- [ ] Testar localmente com tag dummy
- [ ] Validar attestation via `gh attestation verify`
- [ ] Abrir PR com mudanças

### Phase 3: Tag Signature Verification (2-3 dias)
- [ ] **Decisão:** obrigatório ou warning?
- [ ] Documentar política de assinatura em pos-android-hal
- [ ] Implementar step `git verify-tag`
- [ ] Comunicar ao time (como assinar tags)
- [ ] Abrir PR com mudanças

### Phase 4: SBOM Generation (1-2 dias)
- [ ] Adicionar plugin CycloneDX ao pos-android-hal `build.gradle`
- [ ] Implementar steps de build + upload
- [ ] Testar geração de SBOM
- [ ] Validar estrutura JSON
- [ ] Abrir PR com mudanças

### Phase 5: Integration & Documentation (1 dia)
- [ ] Consolidar 3 PRs em um único fluxo
- [ ] Atualizar README com instruções de verificação
- [ ] Comunicar ao time (consumidores)
- [ ] Merge para `main`

**Timeline Estimada:** 1 semana (com reviews paralelos)

---

## Decision Matrix

| Melhoria | Complexidade | Impacto | Prioridade | Bloqueador? |
|---|---|---|---|---|
| SLSA L3 | 🟢 Baixa | 🔴 Alto | 1️⃣ ALTA | Não |
| Tag Signatures | 🟡 Média | 🟡 Médio | 2️⃣ MÉDIA | Team decision |
| SBOM | 🟡 Média | 🟢 Médio | 3️⃣ BAIXA | Não |

**Recomendação:** Fazer em ordem: SLSA L3 → Tag Signatures → SBOM

---

## FAQ & Clarifications

### Q1: O PR #2 precisa esperar por essas mudanças?
**A:** Não! PR #2 já está seguro e pronto para merge. Essas são **melhorias futuras** (nice-to-have). Merge #2 agora, adicione essas melhorias em PRs separados.

### Q2: SLSA L3 é mandatório?
**A:** Para apps de pagamento: **SIM, recomendado**. Para compliance PCI-DSS: **verificar com legal/compliance**.

### Q3: E se alguém não assinar a tag?
**A:** Com a implementação proposta, seria um `warning`, não `error`. Time decide se torna obrigatório.

### Q4: Consumidores precisam fazer algo?
**A:** Para verificar:
```bash
# Verificar SLSA attestation
gh attestation verify ./hal-api-android.aar --owner stone-payments

# Baixar SBOM
gh release download hal-api/3.6.6 --pattern "*.sbom.json"
```

### Q5: Qual é o impacto em CI/CD?
**A:** Mínimo:
- SLSA L3: +5 segundos (assinatura rápida)
- Tag Verification: +2 segundos (git verify-tag)
- SBOM: +10 segundos (gradle plugin)
- **Total:** ~17 segundos adicionais no workflow

---

## References

- 🔗 [SLSA Framework](https://slsa.dev/)
- 🔗 [GitHub Artifact Attestation](https://docs.github.com/en/actions/build-and-test/securing-your-build-artifacts/about-artifact-attestations)
- 🔗 [CycloneDX Gradle Plugin](https://github.com/CycloneDX/cyclonedx-gradle-plugin)
- 🔗 [PCI-DSS Supply Chain Security](https://www.pcisecuritystandards.org/)

---

## Sign-Off Checklist

- [ ] Design aprovado pelo desenvolvedor (você)
- [ ] Design aprovado por SRE/Security
- [ ] Design aprovado pelo team (mobile, payments)
- [ ] Go para Phase 2?
