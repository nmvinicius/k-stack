# Kubernetes Infrastructure Repository

Repositório GitOps para cluster Kubernetes local (Minikube + MetalLB) gerenciado via ArgoCD.

## Arquitetura

Padrão **App-of-Apps** com sync waves para orquestrar a ordem de deploy:

| Sync Wave | Recurso | Descrição |
|-----------|---------|-----------|
| -5 | `AppProject` | Projeto ArgoCD para infraestrutura |
| -3 | `cert-manager` | Helm chart do cert-manager (CRDs + controller) |
| -2 | `cert-manager-configs` | ClusterIssuer self-signed |
| -1 | `gateway-api` | Helm chart NGINX Gateway Fabric |
| 0 | `gateway-api-configs` | Gateways (HTTP/HTTPS/TCP) + Certificado TLS |

## Estrutura do Repositório

```
bootstrap/
└── root-app.yaml                    # Único apply necessário

infrastructure/
├── project.yaml                     # AppProject (wave -5)
├── cert-manager.yaml                # Application Helm (wave -3)
├── cert-manager-configs.yaml        # Application configs (wave -2)
├── gateway-api.yaml                 # Application Helm (wave -1)
├── gateway-api-configs.yaml         # Application configs (wave 0)
├── cert-manager/
│   └── configs/
│       └── self-signed-cluster-issuer.yaml
└── gateway-api/
    ├── README.md
    └── configs/
        ├── http-https-gateway.yaml
        ├── postgres-gateway.yaml
        └── gateway-tls-certificate.yaml

argocd/
└── network-policy.yaml

values/                              # Referência dos values oficiais
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
2. Instalar o cert-manager via Helm
3. Criar o ClusterIssuer self-signed
4. Instalar o NGINX Gateway Fabric via Helm
5. Criar os Gateways (HTTP/HTTPS/TCP) e o certificado TLS

## Componentes

### cert-manager
- Chart: `jetstack/cert-manager` (v1.20.*)
- Gateway API habilitado (`enableGatewayAPI: true`)
- CRDs instalados via Helm
- ClusterIssuer self-signed para certificados locais

### NGINX Gateway Fabric
- Chart: `oci://ghcr.io/nginx/charts/nginx-gateway-fabric` (2.4.2)
- Features experimentais do Gateway API habilitadas
- Gateways configurados: HTTP (80), HTTPS (443), PostgreSQL (5432/5433)
- TLS termination com certificado auto-assinado via cert-manager

---

**Autor:** nmvinicius
