# Kubernetes GitOps — Instruções do Workspace

Repositório GitOps para cluster Kubernetes local (Minikube + MetalLB) gerenciado via ArgoCD.  
Padrão **App-of-Apps**: um único `kubectl apply -f bootstrap/root-app.yaml` bootstrapa toda a infraestrutura.  
Filosofia: **compliance by design** — TLS end-to-end, PKI interna, network policies.

> Veja [README.md](../README.md) para visão geral e [infrastructure/gateway/README.md](../infrastructure/gateway/README.md) para detalhes do NGINX Gateway Fabric.

## Arquitetura

| Sync Wave | Application | Tipo |
|-----------|-------------|------|
| `-5` | `infrastructure` (AppProject) | `AppProject` |
| `-3` | `cert-manager` | Helm chart |
| `-2` | `cert-manager-configs` | Git path |
| `-2` | `gateway-crds` | Kustomize (GitHub) |
| `-1` | `gateway` | Helm chart OCI |
| `-1` | `trust-manager` | Helm chart |
| `0` | `trust-manager-configs` | Git path |
| `0` | `gateway-configs` | Git path |
| `0` | `argocd-configs` | Git path |

**Regra**: CRDs/controllers sempre em wave anterior às configs que os utilizam.

## Bootstrap

```bash
kubectl apply -f bootstrap/root-app.yaml
```

A root app usa `project: default` porque o `AppProject infrastructure` ainda não existe no momento do apply.

## Convenções

### Estrutura de um novo componente

Para adicionar um componente (ex: `stackgres`), seguir este padrão:

```
infrastructure/
├── stackgres.yaml            # Application Helm (wave apropriado)
├── stackgres-configs.yaml    # Application configs (wave +1 após o Helm)
└── stackgres/
    └── configs/
        ├── server-cert.yaml          # Certificate name: stackgres-server
        ├── backend-tls-policy.yaml   # BackendTLSPolicy name: stackgres-server
        └── ...
```

Se o componente precisa de HTTPRoute, adicionar em `infrastructure/gateway/configs/`:

```
infrastructure/gateway/configs/
└── stackgres-httproute.yaml   # HTTPRoute name: stackgres / stackgres-redirect
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
    namespace: <namespace-do-componente>
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

### Valores Helm

- Use `values: |-` (string multiline) para charts que requerem formato de string (ex: `cert-manager`)
- Use `valuesObject:` (YAML estruturado) para charts compatíveis (ex: `gateway`)
- O diretório `values/` contém os values completos dos charts — use como **referência apenas**, não são referenciados pelas Applications

### Nomenclatura

**Resources K8s**: todos os resources de um serviço usam o **mesmo nome base**, permitindo query unificada:

```bash
kubectl get certificate,backendtlspolicy,referencegrant argocd-server -n argocd
```

| Padrão | Exemplo |
|--------|---------|
| Resources de um serviço | mesmo nome base (`argocd-server`) |
| Application Helm | nome do componente (`cert-manager`, `gateway`) |
| Application de configs | sufixo `-configs` (`cert-manager-configs`) |
| Resources padrão do app | manter nome original (`argocd-cmd-params-cm`) |
| Secrets TLS | `<recurso>-tls` (`gateway-tls`) |
| Gateways | descreve protocolos (`http-https-gateway`, `postgres-gateway`) |
| Arquivos de configs | descrevem conteúdo, sem prefixo redundante (`server-cert.yaml`, não `argocd-server-cert.yaml`) |

## Armadilhas Comuns

- **`ignoreDifferences`**: necessário para recursos que sofrem drift gerenciado externamente (ex: `cert-generator` do gateway). Adicionar quando um resource controller sobrescreve campos constantemente.
- **Ordem de sources**: ao usar OCI registries como `repoURL`, o campo `chart` é obrigatório; não usar `path`.
- **AppProject antes de tudo**: qualquer nova Application deve usar `project: infrastructure`; só a root app usa `project: default`.
- **`recurse: false`** na root app: o diretório `infrastructure/` é lido sem recursão — subdiretórios (`cert-manager/configs/`) são gerenciados por Applications separadas.
- **ReferenceGrant**: fica no namespace do destino (ex: `argocd/configs/reference-grant.yaml`), não no namespace do gateway.

## Agentes disponíveis

| Arquivo | Chamada | Uso |
|---------|---------|-----|
| `agents/conventional-commit.agent.md` | `/conventional-commit` | Cria commits convencionais analisando o diff |
| `agents/kubectl-explain-skill.md` | `/kubectl-explain` | Explora campos de recursos Kubernetes |
| `agents/argocd-review.agent.md` | `/argocd-review` | Revisa saúde e sync das Applications via MCP ArgoCD |
