# Sync Waves (ArgoCD)

Este arquivo define os valores de `argocd.argoproj.io/sync-wave` usados no repositório e o critério para validar se estao corretos.

## Mapa Atual (validado)

| Wave | Recurso | Arquivo | Status |
|------|---------|---------|--------|
| `-5` | `AppProject infrastructure` | `infrastructure/project/application.yaml` | Correto |
| `-3` | `Application cert-manager` | `infrastructure/cert-manager/application.yaml` | Correto |
| `-2` | `Application gateway` | `infrastructure/gateway/application.yaml` | Correto |
| `-1` | `Application trust-manager` | `infrastructure/trust-manager/application.yaml` | Correto |
| `0` | `Application argocd` | `infrastructure/argocd/application.yaml` | Correto |
| `1` | `Application prometheus` | `infrastructure/prometheus/application.yaml` | Correto |
| `2` | `Application stackgres` | `infrastructure/stackgres/application.yaml` | Correto |

## Regra de Ordenacao

1. Infraestrutura base primeiro (projeto ArgoCD): `-5`.
2. Controllers e CRDs antes dos consumidores:
   - `cert-manager` antes de recursos que pedem certificados.
   - `gateway` antes de `HTTPRoute`, `BackendTLSPolicy` e `ReferenceGrant`.
   - `trust-manager` antes de workloads que validam CA por bundle.
3. Aplicacoes de plataforma e observabilidade depois da base:
   - `argocd` em `0`.
   - `prometheus` em `1`.
   - `stackgres` em `2`.

## Regra Dentro de Cada Application

- Recursos em `infrastructure/<app>/configs/**` devem usar `sync-wave: "1"`.
- Motivo: em app multi-source (Helm + Git), o chart sobe primeiro e as configs aplicam depois, quando controller/CRDs ja estao disponiveis.

## Checklist de Validacao Rapida

1. Todo `Application` e `AppProject` possui annotation `argocd.argoproj.io/sync-wave`.
2. A sequencia global nao quebra dependencias de CRD/controller.
3. Recursos de `configs/` ficam em `sync-wave: "1"`.
4. `bootstrap/root-app.yaml` descobre apenas `**/application.yaml`.

## Comando de Auditoria

```bash
rg -n 'argocd\.argoproj\.io/sync-wave:\s*"[-0-9]+"' bootstrap infrastructure | sort
```

Se adicionar novo componente, atualizar este arquivo no mesmo PR.
