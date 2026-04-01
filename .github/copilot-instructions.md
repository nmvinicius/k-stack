# Kubernetes GitOps — Instruções do Workspace

Repositório GitOps para cluster Kubernetes local (Minikube + MetalLB) gerenciado via ArgoCD.  
Padrão **App-of-Apps**: um único `kubectl apply -f bootstrap/root-app.yaml` bootstrapa toda a infraestrutura.  
Filosofia: **compliance by design** — TLS end-to-end, PKI interna, network policies.

> Veja [README.md](../README.md) para visão geral do projeto.

## Arquitetura

| Sync Wave | Application | Tipo |
|-----------|-------------|------|
| `-5` | `infrastructure` (AppProject) | `AppProject` |
| `-3` | `cert-manager` | Multi-source (Helm + Git) |
| `-2` | `gateway-crds` | Git path (GitHub) |
| `-1` | `gateway` | Multi-source (Helm OCI + Git) |
| `-1` | `trust-manager` | Multi-source (Helm + Git) |
| `0` | `argocd` | Git path |

**Regra**: CRDs/controllers sempre em wave anterior às configs que os utilizam.
**Multi-source**: Helm + configs no mesmo Application; configs usam `sync-wave: "1"` para garantir que o controller esteja healthy.

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
├── stackgres.yaml            # Application multi-source (wave apropriado)
└── stackgres/
    └── configs/
        └── stackgres-server/         # Pasta = conceito/serviço
            ├── certificate.yaml      # Kind como nome do arquivo
            ├── backend-tls-policy.yaml
            └── reference-grant.yaml  # Todos com sync-wave: "1"
```

Se o componente precisa de HTTPRoute, adicionar em `infrastructure/gateway/configs/`:

```
infrastructure/gateway/configs/
└── stackgres/
    ├── httproute.yaml            # Kind padrão — sem sufixo
    └── httproute-redirect.yaml   # Variante — com sufixo
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
  sources:                                    # Multi-source
    - chart: <chart-name>                     # Helm source
      repoURL: <registry>
      targetRevision: <version>
      helm:
        valuesObject: {}
    - repoURL: git@github.com:nmvinicius/k-stack.git
      targetRevision: HEAD
      path: infrastructure/<componente>/configs
      directory:
        recurse: true
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

Resources dentro de `configs/` devem ter `argocd.argoproj.io/sync-wave: "1"` para aplicar após o Helm chart.

### Valores Helm

- Use `values: |-` (string multiline) para charts que requerem formato de string (ex: `cert-manager`)
- Use `valuesObject:` (YAML estruturado) para charts compatíveis (ex: `gateway`)
- Para consultar os values padrão de um chart: `helm show values <chart>`

### Nomenclatura

**Resources K8s**: todos os resources de um serviço usam o **mesmo nome base**, permitindo query unificada:

```bash
kubectl get certificate,backendtlspolicy,referencegrant argocd-server -n argocd
```

**Estrutura de pastas configs/**: concept folders + nomes de arquivo por Kind:

```
configs/
└── <conceito>/              # resource name ou agrupamento lógico
    ├── certificate.yaml     # Kind padrão (leaf cert) — sem sufixo
    ├── certificate-ca.yaml  # Variante (CA cert) — com sufixo
    ├── secret.yaml          # Opaque (padrão) — sem sufixo
    └── secret-tls.yaml      # TLS variant — com sufixo
```

| Padrão | Exemplo |
|--------|---------|
| Pasta de configs | nome do conceito/serviço (`argocd-server/`, `ngf-internal-tls/`) |
| Arquivo | Kind em kebab-case (`certificate.yaml`, `backend-tls-policy.yaml`) |
| Sufixo de variante | tipo não-padrão (`certificate-ca.yaml`, `httproute-redirect.yaml`) |
| Resources de um serviço | mesmo nome base (`argocd-server`) |
| Application | nome do componente (`cert-manager`, `gateway`, `argocd`) |
| Gateways | descreve protocolos (`http-https-gateway`, `postgres-gateway`) |

## Armadilhas Comuns

- **`ignoreDifferences`**: necessário para recursos que sofrem drift gerenciado externamente (ex: `cert-generator` do gateway). Adicionar quando um resource controller sobrescreve campos constantemente.
- **Ordem de sources**: ao usar OCI registries como `repoURL`, o campo `chart` é obrigatório; não usar `path`.
- **AppProject antes de tudo**: qualquer nova Application deve usar `project: infrastructure`; só a root app usa `project: default`.
- **`recurse: false`** na root app: o diretório `infrastructure/` é lido sem recursão — subdiretórios (`cert-manager/configs/`) são gerenciados via `sources` das Applications.
- **`directory.recurse: true`** nas sources Git das Applications: necessário para ler concept folders dentro de `configs/`.
- **`sync-wave: "1"`** em resources de configs: garante que o controller do Helm chart esteja healthy antes de aplicar configs (ClusterIssuers, Certificates, etc.).
- **ReferenceGrant**: fica no namespace do destino (ex: `argocd/configs/argocd-server/reference-grant.yaml`), não no namespace do gateway.

## Agentes disponíveis

| Arquivo | Chamada | Uso |
|---------|---------|-----|
| `agents/conventional-commit.agent.md` | `/conventional-commit` | Cria commits convencionais analisando o diff |
| `agents/kubectl-explain-skill.md` | `/kubectl-explain` | Explora campos de recursos Kubernetes |
| `agents/argocd-review.agent.md` | `/argocd-review` | Revisa saúde e sync das Applications via MCP ArgoCD |
