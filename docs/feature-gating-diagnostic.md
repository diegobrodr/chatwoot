# Diagnóstico técnico de travas de recursos (sem bypass)

> Objetivo: mapear onde o Chatwoot aplica bloqueios por plano/licença no código, para auditoria e planejamento de alternativas legítimas.

## 1) Origem principal das travas por recurso

A base de features fica em `config/features.yml`. Cada feature pode ter `enabled`, `premium`, `chatwoot_internal` e `deprecated`.

- Recursos marcados com `premium: true` no snapshot atual:
  - `disable_branding`
  - `audit_logs`
  - `response_bot`
  - `sla`
  - `help_center_embedding_search`
  - `captain_integration`
  - `custom_roles`
  - `channel_voice`
  - `captain_integration_v2`
  - `advanced_search`
  - `saml`
  - `advanced_search_indexing`
  - `companies`
  - `csat_review_notes`
  - `conversation_required_attributes`
  - `advanced_assignment`

## 2) Como o backend aplica gate por feature

### 2.1 Flags por conta

- O concern `Featurable` carrega `config/features.yml` e expõe `feature_enabled?(name)`.
- Em runtime, as validações de acesso são feitas principalmente por `account.feature_enabled?(...)`.

### 2.2 Edição/instância

- `ChatwootApp.enterprise?` detecta se o diretório `enterprise/` existe e pode ser desativado por `DISABLE_ENTERPRISE`.
- `ChatwootHub.pricing_plan` retorna `community` quando não for enterprise, e no EE lê `INSTALLATION_PRICING_PLAN`.

### 2.3 Reconciliação de plano (EE)

No overlay Enterprise, existe reconciliação periódica para instalações `community`:

- `Internal::ReconcilePlanConfigService`:
  - Reaplica valores de branding de `enterprise/config/premium_installation_config.yml`.
  - Desabilita features premium em todas as contas com `account.disable_features!(*premium_features)` usando `enterprise/config/premium_features.yml`.

## 3) Onde aparecem paywalls/upsell no frontend

- `app/javascript/dashboard/featureFlags.js` mantém um subconjunto `PREMIUM_FEATURES` para UI.
- `usePolicy.js` usa:
  - `shouldShow(...)`: permite renderizar item para poder mostrar upsell/paywall mesmo quando desabilitado.
  - `shouldShowPaywall(...)`: decide quando exibir paywall (Cloud/Enterprise sem plano premium).

## 4) Gating por limites de uso (EE)

- Em OSS, `Account#usage_limits` retorna limites altos padrão (`ChatwootApp.max_limit`).
- Em Enterprise, `Enterprise::Account::PlanUsageAndLimits` sobrescreve `usage_limits` e combina:
  - limites por assinatura (`subscribed_quantity`),
  - `account.limits`,
  - configs globais (`ACCOUNT_*_LIMIT`),
  - cotas Captain por plano (`CAPTAIN_CLOUD_PLAN_LIMITS`) com consumo (`captain_*_usage`).

## 5) Exemplos de pontos de bloqueio (código de produto)

- Busca avançada: `SearchService` e `Message` verificam `advanced_search`/`advanced_search_indexing`.
- SAML: controllers EE verificam `Current.account.feature_enabled?('saml')`.
- SLA/Assignment/Captain: vários serviços/finders EE exigem flags específicas e limites disponíveis.
- Branding no widget/portal: usa `feature_enabled?('disable_branding')`.

## 6) Alternativas legítimas (sem remover licenciamento)

1. **Configuração oficial do plano/licença**
   - Validar `INSTALLATION_PRICING_PLAN` e sincronização do Hub para habilitar recursos pagos de forma suportada.
2. **Uso apenas de recursos community/OSS**
   - Ativar apenas features não premium por conta e adaptar fluxos.
3. **Extensão custom em `custom/`**
   - Implementar funcionalidades próprias sem alterar mecanismos de licenciamento do produto.
4. **Governança de configurações**
   - Revisar `installation_config.yml` e chaves globais para não depender de valores premium reconciliados.

## 7) Comandos úteis para auditoria contínua

```bash
rg -n "premium: true" config/features.yml enterprise/config/premium_features.yml
rg -n "feature_enabled\?|PREMIUM_FEATURES|shouldShowPaywall|pricing_plan|ReconcilePlanConfigService" app enterprise lib
rg -n "usage_limits|CAPTAIN_CLOUD_PLAN_LIMITS|ACCOUNT_.*_LIMIT" app enterprise config
```

---

Este documento é intencionalmente de diagnóstico e conformidade. Não inclui passos para bypass de licença/pagamento.
