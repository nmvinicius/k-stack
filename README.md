# Kubernetes Infrastructure Repository

Repositório GitOps para cluster Kubernetes local (Minikube + MetalLB) gerenciado via ArgoCD.
Compliance by design — TLS end-to-end, PKI interna, network policies.

## Arquitetura

Padrão **App-of-Apps** com sync waves para orquestrar a ordem de deploy:

| Sync Wave | Application | Descrição |
|-----------|-------------|-----------|
| `-5` | `infrastructure` | AppProject ArgoCD |
| `-3` | `cert-manager` | cert-manager + PKI (multi-source) |
| `-2` | `gateway` | CRDs + NGINX Gateway Fabric + configs (multi-source) |
| `-1` | `trust-manager` | trust-manager + CA bundle (multi-source) |
| `0` | `argocd` | Certificates, BackendTLSPolicy, NetworkPolicy |
| `1` | `prometheus` | kube-prometheus-stack + configs (multi-source) |
| `2` | `stackgres` | stackgres-operator + configs (multi-source) |

Mapeamento e criterio de validacao dos waves: `SYNC-WAVES.md`.

## Estrutura do Repositório

```
bootstrap/
└── root-app.yaml                        # Único apply necessário

infrastructure/
├── project/
│   └── application.yaml                 # AppProject (wave -5)
│
├── cert-manager/
│   ├── application.yaml                 # Multi-source: Helm + configs (wave -3)
│   └── configs/
│       ├── selfsigned-issuer/
│       │   └── cluster-issuer.yaml
│       └── cluster-internal-ca/
│           ├── certificate-ca.yaml
│           └── cluster-issuer.yaml
│
├── gateway/
│   ├── application.yaml                 # Multi-source: CRDs + Helm + configs (wave -2)
│   └── configs/
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
├── trust-manager/
│   ├── application.yaml                 # Multi-source: Helm + configs (wave -1)
│   └── configs/
│       └── cluster-internal-ca/
│           └── bundle.yaml
│
├── argocd/
│   ├── application.yaml                 # Configs (wave 0)
│   └── configs/
│       ├── argocd-server/
│       │   ├── httproute.yaml
│       │   ├── httproute-redirect.yaml
│       │   ├── certificate.yaml
│       │   ├── backend-tls-policy.yaml
│       │   └── reference-grant.yaml
│       └── repo-server/
│           └── network-policy.yaml
│
├── prometheus/
│   ├── application.yaml
│   └── configs/
│       ├── grafana/
│       │   ├── certificate.yaml
│       │   ├── backend-tls-policy.yaml
│       │   ├── reference-grant.yaml
│       │   ├── httproute.yaml
│       │   └── httproute-redirect.yaml
│       └── nginx-gateway-fabric/
│           └── pod-monitor.yaml
│
└── stackgres/
    ├── application.yaml
    └── configs/
        └── stackgres-operator/
            ├── certificate.yaml
            ├── backend-tls-policy.yaml
            ├── reference-grant.yaml
            ├── httproute.yaml
            └── httproute-redirect.yaml
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

A root app varre `infrastructure/` recursivamente, mas filtra apenas `**/application.yaml`.

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
- `gateway/configs` contém apenas recursos do domínio gateway (Gateways, TLS e PKI interna do NGF)

### ArgoCD
- TLS re-encryption: Gateway → TLS → argocd-server
- BackendTLSPolicy valida cert com CA interno
- Network policy para applicationset-controller → repo-server
- HTTPRoutes ficam em `argocd/configs/argocd-server/` para manter isolamento por app

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
