# Dk8s – Descomplicando Kubernetes

Repositório usado para consolidar os estudos do curso Dk8s – Descomplicando Kubernetes. Aqui ficam os manifestos, anotações e comandos executados ao longo das aulas, permitindo reproduzir o ambiente de laboratório e revisar os conceitos vistos.

## Day-1 — Introdução ao Kubernetes

### 1. Conhecimentos adquiridos
- O que é Kubernetes e por que orquestradores são necessários para aplicações em containers.
- Arquitetura básica: Control Plane (API Server, etcd, scheduler, controller manager) e Nodes que rodam kubelet e kube-proxy.
- Conceitos de container, container runtime e container engine; papel do Docker/Containerd no ciclo de vida dos containers.
- Estrutura de um cluster, comunicação via API Server e como os componentes interagem.
- Diferença entre TCP x UDP e impacto em exposição de serviços.
- Objetos fundamentais: Pods, ReplicaSets, Deployments e Services, e quando utilizar cada um.
- Visão geral de networking no Kubernetes: redes internas, IPs dos Pods/Services e roteamento dentro do cluster.

### 2. Ferramentas utilizadas

| Ferramenta | Uso no laboratório |
| --- | --- |
| kind (Kubernetes in Docker) | Criação de cluster local para testes rápidos. |
| kubectl | Cliente para aplicar manifestos, inspecionar e depurar recursos. |
| WSL2 | Ambiente Linux leve para executar Docker e kubectl no Windows. |
| Docker / Containerd | Engine/runtime responsável por criar containers usados pelo Kind. |

### 3. Atividades práticas realizadas
- Criação de um cluster local com Kind a partir de um arquivo de configuração.
- Validação do cluster e dos nós (`kubectl get nodes`).
- Criação de Pods e Deployments.
- Criação de Services do tipo NodePort.
- Exposição de Pods e consumo via NodePort e `kubectl port-forward`.
- Entendimento dos IPs internos criados pelo Kind (rede Docker) e teste de conectividade.
- Execução de comandos dentro do Pod com `kubectl exec`.
- Uso de labels e selectors para expor workloads com `kubectl expose`.

### 4. Estrutura de diretórios
```
.
└── day-1/
    └── kind/
        ├── kind-cluster.yaml
        └── pod.yaml
```

### 5. Comandos executados no Day-1
```bash
# Criar cluster local com configuração personalizada
kind create cluster --config day-1/kind/kind-cluster.yaml

# Verificar nós e componentes básicos
kubectl get nodes
kubectl get pods -A

# Criar Deployment/Pod (exemplo com nginx)
kubectl create deployment nhs-app --image=nginx
kubectl apply -f day-1/kind/pod.yaml

# Expor aplicação
kubectl expose deployment nhs-app --type=NodePort --port=80
kubectl get svc

# Acessar serviços
kubectl port-forward deployment/nhs-app 8080:80
kubectl exec -it pod/nhs-app-2 -- sh

# Limpeza (opcional)
kind delete cluster
```

### 6. Conclusão do dia
- Foi construído um entendimento sólido sobre a base do Kubernetes: arquitetura, objetos primários e fluxo de comunicação.
- Ficou claro como containers são empacotados e executados em Pods, e como Services (especialmente NodePort) expõem workloads.
- As práticas focaram em criar e inspecionar recursos via kubectl, reforçando a interação com a API Server.
- Próximos passos: aprofundar em Deployments completos (réplicas, rollouts), Services avançados (ClusterIP, LoadBalancer), ConfigMaps/Secrets, volumes e políticas de rede.
