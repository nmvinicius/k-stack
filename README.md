# Kubernetes Infrastructure Repository

Este repositório contém a configuração de infraestrutura para um cluster Kubernetes utilizando o ArgoCD.

## Estrutura do Repositório

- **bootstrap/**
  - `project.yaml`: Define o projeto ArgoCD para gerenciar ferramentas de infraestrutura.
  - `root-app.yaml`: Define a aplicação raiz que sincroniza toda a infraestrutura.

- **infrastructure/cert-manager/**
  - `application.yaml`: Configuração da aplicação `cert-manager` no ArgoCD.
  - `values.yaml`: Valores padrão para o Helm Chart do `cert-manager`.

## Pré-requisitos

- Kubernetes cluster configurado.
- ArgoCD instalado e configurado no cluster.
- Acesso ao repositório Git via SSH.

## Configuração

1. **Configurar o acesso SSH**:
   - Certifique-se de que sua chave SSH está configurada corretamente.
   - Adicione a chave pública ao repositório Git, se necessário.

2. **Aplicar as configurações**:
   - Sincronize o projeto e as aplicações no ArgoCD:
     ```bash
     argocd app create -f bootstrap/project.yaml
     argocd app create -f bootstrap/root-app.yaml
     ```

3. **Validar a instalação**:
   - Verifique se o `cert-manager` foi instalado corretamente:
     ```bash
     kubectl get pods -n cert-manager
     ```

## Contribuição

- Sinta-se à vontade para abrir issues ou pull requests para melhorias.

---

**Autor:** nmvinicius
