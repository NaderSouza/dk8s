# Headlamp — UI para visualizar o cluster

[Headlamp](https://headlamp.dev) é a interface web oficial do SIG UI do Kubernetes. Neste repo ele serve para **enxergar em tempo real** o que os manifestos das aulas criam no cluster kind: pods, deployments, services, eventos, logs e o YAML de cada objeto.

Ele lê o estado direto da API do cluster, então mostra tudo que foi aplicado com `kubectl apply`, sem precisar de commit. É diferente de uma ferramenta GitOps como o Argo CD, que só mostra (e aplica) o que está no Git.

---

## Pré-requisitos

| Ferramenta | Para quê | Como conferir |
| --- | --- | --- |
| kind | Cluster local | `kind get clusters` (deve listar `dk8s`) |
| kubectl | Cliente do cluster | `kubectl config current-context` (deve ser `kind-dk8s`) |
| helm 3 | Instalar o chart | `helm version --short` |

Se o cluster ainda não existe:

```bash
kind create cluster --name dk8s --config day-1/kind/kind-cluster.yaml
```

---

## Passo a passo

### 1. Adicionar o repositório do chart

```bash
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
helm repo update headlamp
```

### 2. Instalar o Headlamp no cluster

```bash
helm upgrade --install headlamp headlamp/headlamp \
  --kube-context kind-dk8s \
  -n headlamp --create-namespace \
  --wait
```

- `upgrade --install` serve tanto para a primeira instalação quanto para atualizar depois.
- `--kube-context kind-dk8s` garante que a instalação vai para o cluster de estudo, mesmo que o contexto ativo seja outro.
- O chart cria a ServiceAccount `headlamp` e a liga ao `cluster-admin` (ClusterRoleBinding `headlamp-admin`).

### 3. Conferir a instalação

```bash
kubectl --context kind-dk8s -n headlamp get pods,svc
```

O pod `headlamp-...` deve estar `Running`, e o Service `headlamp` do tipo `ClusterIP` na porta 80.

### 4. Abrir o acesso local

Deixe este comando rodando num terminal:

```bash
kubectl --context kind-dk8s -n headlamp port-forward svc/headlamp 4466:80
```

### 5. Gerar o token de login

Em outro terminal:

```bash
kubectl --context kind-dk8s create token headlamp -n headlamp --duration 24h
```

O token é um JWT da ServiceAccount `headlamp` e expira em 24h. Para entrar de novo depois disso, gere outro.

### 6. Entrar na UI

Abra **http://localhost:4466** no navegador. No WSL2 o navegador do Windows alcança o `localhost` do Linux. Cole o token do passo 5.

---

## Validação

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:4466/   # 200
```

Teste prático: com a UI aberta, aplique um manifesto de uma aula e veja os objetos aparecerem sem recarregar a página.

```bash
kubectl apply -f day-4/nginx-deployment.yaml
```

---

## Problemas comuns

| Sintoma | Causa provável | Solução |
| --- | --- | --- |
| `localhost:4466` não abre | O `port-forward` parou (terminal fechado, pod reiniciado) | Rodar o passo 4 de novo |
| Tela de login recusa o token | Token expirado ou gerado em outro cluster | Gerar outro (passo 5) com `--context kind-dk8s` |
| `Error: Kubernetes cluster unreachable` no helm | Cluster kind parado ou contexto errado | `kind get clusters` e `docker ps` para ver se os nós estão de pé |
| Headlamp sumiu | O cluster foi recriado com `kind delete/create` | Repetir os passos 2, 4 e 5 |

---

## Segurança

`cluster-admin` via token é aceitável **apenas** neste cluster local de estudo. Em clusters reais, use uma ServiceAccount com RBAC mínimo (Role por namespace) ou login via OIDC.

---

## Remover

```bash
helm uninstall headlamp --kube-context kind-dk8s -n headlamp
kubectl --context kind-dk8s delete namespace headlamp
```
