# Kubernetes Infrastructure Repository

Repositório GitOps para cluster Kubernetes local (Minikube + MetalLB) gerenciado via ArgoCD.
Compliance by design — TLS end-to-end, PKI interna, network policies.

## Arquitetura

Padrão **App-of-Apps** com sync waves para orquestrar a ordem de deploy:

| Sync Wave | Application | Descrição |
|-----------|-------------|-----------|
| `-5` | `infrastructure` | AppProject ArgoCD |
| `-3` | `cert-manager` | Helm chart cert-manager (CRDs + controller) |
| `-2` | `cert-manager-configs` | ClusterIssuer self-signed + CA interno |
| `-2` | `gateway-crds` | CRDs do Gateway API (experimental) |
| `-1` | `gateway` | Helm chart NGINX Gateway Fabric |
| `-1` | `trust-manager` | Helm chart trust-manager |
| `0` | `trust-manager-configs` | Bundle de distribuição do CA |
| `0` | `gateway-configs` | Gateways (HTTP/HTTPS/TCP) + TLS + HTTPRoutes |
| `0` | `argocd-configs` | Certificates, BackendTLSPolicy, NetworkPolicy |

## Estrutura do Repositório

```
bootstrap/
└── root-app.yaml                        # Único apply necessário

infrastructure/
├── project.yaml                         # AppProject (wave -5)
│
├── cert-manager.yaml                    # Application Helm (wave -3)
├── cert-manager-configs.yaml            # Application configs (wave -2)
├── cert-manager/
│   └── configs/
│       ├── self-signed-cluster-issuer.yaml
│       └── cluster-internal-ca.yaml
│
├── trust-manager.yaml                   # Application Helm (wave -1)
├── trust-manager-configs.yaml           # Application configs (wave 0)
├── trust-manager/
│   └── configs/
│       └── cluster-ca-bundle.yaml
│
├── gateway-crds.yaml                    # Application CRDs (wave -2)
├── gateway.yaml                         # Application Helm (wave -1)
├── gateway-configs.yaml                 # Application configs (wave 0)
├── gateway/
│   ├── README.md
│   └── configs/
│       ├── http-https-gateway.yaml
│       ├── postgres-gateway.yaml
│       ├── gateway-tls.yaml
│       ├── ngf-internal-tls.yaml
│       └── argocd-httproute.yaml
│
├── argocd-configs.yaml                  # Application configs (wave 0)
└── argocd/
    └── configs/
        ├── server-cert.yaml
        ├── backend-tls-policy.yaml
        ├── reference-grant.yaml
        ├── cmd-params.yaml
        └── network-policy.yaml

values/                                  # Referência dos values oficiais
├── cert-manager.yaml
└── gateway-api.yaml
```

## Pré-requisitos

- Minikube com MetalLB configurado
- ArgoCD instalado no cluster
- Acesso SSH ao repositório Git

## Como usar

Um único comando bootstrapa toda a infraestrutura:

```bash
kubectl apply -f bootstrap/root-app.yaml
```

O ArgoCD irá, em ordem:
1. Criar o AppProject `infrastructure`
2. Instalar cert-manager e CRDs do Gateway API
3. Criar ClusterIssuers e CA interno
4. Instalar NGINX Gateway Fabric e trust-manager
5. Configurar Gateways, TLS, HTTPRoutes e distribuir o CA bundle
6. Aplicar configs do ArgoCD (certificate, TLS policy, network policy)

## Componentes

### cert-manager
- Chart: `jetstack/cert-manager` (v1.20.*)
- Gateway API habilitado (`enableGatewayAPI: true`)
- CRDs instalados via Helm
- PKI: ClusterIssuer self-signed → CA Certificate → ClusterIssuer CA

### trust-manager
- Chart: `jetstack/trust-manager` (0.14.0)
- Distribui o CA cert interno como ConfigMap em todos os namespaces

### NGINX Gateway Fabric
- Chart: `oci://ghcr.io/nginx/charts/nginx-gateway-fabric` (2.4.2)
- Features experimentais do Gateway API habilitadas
- Gateways: HTTP (80), HTTPS (443), PostgreSQL (5432/5433)
- TLS termination com certificado wildcard `*.k8s.local`

### ArgoCD
- TLS re-encryption: Gateway → TLS → argocd-server
- BackendTLSPolicy valida cert com CA interno
- Network policy para applicationset-controller → repo-server

## Convenção de nomes

Resources de um mesmo serviço compartilham o **mesmo nome base** para facilitar queries:

```bash
kubectl get certificate,backendtlspolicy,referencegrant argocd-server -n argocd
```

| Padrão | Exemplo |
|--------|---------|
| Resources de um serviço | mesmo nome (`argocd-server`) |
| Application Helm | nome do componente (`cert-manager`, `gateway`) |
| Application de configs | sufixo `-configs` (`cert-manager-configs`) |
| Resources padrão do app | manter nome original (`argocd-cmd-params-cm`) |

---

**Autor:** nmvinicius
