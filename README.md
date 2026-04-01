# Kubernetes Infrastructure Repository

Repositório GitOps para cluster Kubernetes local (Minikube + MetalLB) gerenciado via ArgoCD.
Compliance by design — TLS end-to-end, PKI interna, network policies.

## Arquitetura

Padrão **App-of-Apps** com sync waves para orquestrar a ordem de deploy:

| Sync Wave | Application | Descrição |
|-----------|-------------|-----------|
| `-5` | `infrastructure` | AppProject ArgoCD |
| `-3` | `cert-manager` | cert-manager + PKI (multi-source) |
| `-2` | `gateway-crds` | CRDs do Gateway API (experimental) |
| `-1` | `gateway` | NGINX Gateway Fabric + configs (multi-source) |
| `-1` | `trust-manager` | trust-manager + CA bundle (multi-source) |
| `0` | `argocd` | Certificates, BackendTLSPolicy, NetworkPolicy |

## Estrutura do Repositório

```
bootstrap/
└── root-app.yaml                        # Único apply necessário

infrastructure/
├── project.yaml                         # AppProject (wave -5)
│
├── cert-manager.yaml                    # Multi-source: Helm + configs (wave -3)
├── cert-manager/
│   └── configs/
│       ├── selfsigned-issuer/
│       │   └── cluster-issuer.yaml
│       └── cluster-internal-ca/
│           ├── certificate-ca.yaml
│           └── cluster-issuer.yaml
│
├── gateway-crds.yaml                    # CRDs (wave -2)
├── gateway.yaml                         # Multi-source: Helm + configs (wave -1)
├── gateway/
│   └── configs/
│       ├── argocd/
│       │   ├── httproute.yaml
│       │   └── httproute-redirect.yaml
│       ├── gateway-tls/
│       │   └── certificate.yaml
│       ├── http-https-gateway/
│       │   └── gateway.yaml
│       ├── ngf-internal-tls/
│       │   ├── certificate-ca.yaml
│       │   ├── certificate.yaml
│       │   └── issuer.yaml
│       └── postgres-gateway/
│           └── gateway.yaml
│
├── trust-manager.yaml                   # Multi-source: Helm + configs (wave -1)
├── trust-manager/
│   └── configs/
│       └── cluster-internal-ca/
│           └── bundle.yaml
│
├── argocd.yaml                          # Configs (wave 0)
└── argocd/
    └── configs/
        ├── argocd-server/
        │   ├── certificate.yaml
        │   ├── backend-tls-policy.yaml
        │   └── reference-grant.yaml
        └── repo-server/
            └── network-policy.yaml
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
2. Instalar cert-manager + criar PKI (ClusterIssuers, CA)
3. Instalar CRDs do Gateway API
4. Instalar NGINX Gateway Fabric + Gateways, TLS, HTTPRoutes
5. Instalar trust-manager + distribuir CA bundle
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
| Application | nome do componente (`cert-manager`, `gateway`, `argocd`) |
| Resources padrão do app | manter nome original (`argocd-cmd-params-cm`) |

---

**Autor:** nmvinicius
