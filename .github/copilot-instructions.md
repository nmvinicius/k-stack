# Kubernetes GitOps — Instruções do Workspace

Repositório GitOps para cluster Kubernetes local (Minikube + MetalLB) gerenciado via ArgoCD.  
Padrão **App-of-Apps**: um único `kubectl apply -f bootstrap/root-app.yaml` bootstrapa toda a infraestrutura.

> Veja [README.md](../README.md) para visão geral e [infrastructure/gateway-api/README.md](../infrastructure/gateway-api/README.md) para detalhes do NGINX Gateway Fabric.

## Arquitetura

| Sync Wave | Application | Tipo |
|-----------|-------------|------|
| `-5` | `infrastructure` (AppProject) | `AppProject` |
| `-3` | `cert-manager` | Helm chart |
| `-2` | `cert-manager-configs` | Git path |
| `-2` | `gateway-api-crds` | Kustomize (GitHub) |
| `-1` | `gateway-api` | Helm chart OCI |
| `0` | `gateway-api-configs` | Git path |

**Regra**: CRDs/controllers sempre em wave anterior às configs que os utilizam.

## Bootstrap

```bash
kubectl apply -f bootstrap/root-app.yaml
```

A root app usa `project: default` porque o `AppProject infrastructure` ainda não existe no momento do apply.

## Convenções

### Estrutura de um novo componente

Para adicionar um componente (ex: `monitoring`), seguir este padrão:

```
infrastructure/
├── monitoring.yaml          # Application Helm (wave apropriado)
├── monitoring-configs.yaml  # Application configs (wave +1 após o Helm)
└── monitoring/
    └── configs/
        └── *.yaml           # Manifests gerenciados pelo monitoring-configs
```

### Template de Application ArgoCD

Todo `Application` deve ter:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <nome>
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "<número>"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: infrastructure
  destination:
    namespace: <mesmo-nome-do-componente>
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

### Valores Helm

- Use `values: |-` (string multiline) para charts que requerem formato de string (ex: `cert-manager`)
- Use `valuesObject:` (YAML estruturado) para charts compatíveis (ex: `gateway-api`)
- O diretório `values/` contém os values completos dos charts — use como **referência apenas**, não são referenciados pelas Applications

### Nomenclatura

| Padrão | Exemplo |
|--------|---------|
| Application Helm | nome do chart (`cert-manager`) |
| Application de configs | sufixo `-configs` (`cert-manager-configs`) |
| Namespace de destino | mesmo nome da Application |
| Secrets TLS | `<recurso>-tls` (`gateway-tls`) |
| Gateways | descreve protocolos (`http-https-gateway`, `postgres-gateway`) |

## Armadilhas Comuns

- **`ignoreDifferences`**: necessário para recursos que sofrem drift gerenciado externamente (ex: `cert-generator` do gateway-api). Adicionar quando um resource controller sobrescreve campos constantemente.
- **Ordem de sources**: ao usar OCI registries como `repoURL`, o campo `chart` é obrigatório; não usar `path`.
- **AppProject antes de tudo**: qualquer nova Application deve usar `project: infrastructure`; só a root app usa `project: default`.
- **`recurse: false`** na root app: o diretório `infrastructure/` é lido sem recursão — subdiretórios (`cert-manager/configs/`) são gerenciados por Applications separadas.

## Agentes disponíveis

| Arquivo | Chamada | Uso |
|---------|---------|-----|
| `agents/conventional-commit.agent.md` | `/conventional-commit` | Cria commits convencionais analisando o diff |
| `agents/kubectl-explain-skill.md` | `/kubectl-explain` | Explora campos de recursos Kubernetes |
| `agents/argocd-review.agent.md` | `/argocd-review` | Revisa saúde e sync das Applications via MCP ArgoCD |
