# Desafio II - Fundamentos de Kubernetes

Este repositório contém a solução completa para o **Desafio II de Fundamentos de Kubernetes**. O projeto baseia-se na criação, configuração e orquestração de uma infraestrutura *multi-tier* (Base de Dados + API REST) num cluster Kubernetes local (K3s no Fedora Linux). 

Todo o desenvolvimento seguiu as melhores práticas de Infraestrutura como Código (IaC) e o fluxo de versionamento **GitFlow**.

---

## Arquitetura do Projeto

A aplicação é composta por dois componentes principais a correr de forma isolada no namespace `segundodesafio`:

1. **PostgreSQL (Database - Stateful):** 
   - Armazenamento persistente configurado via `PersistentVolumeClaim` (PVC), garantindo que os dados não sejam perdidos caso o Pod seja recriado.
   - Comunicação interna ativada através de um `Service` do tipo ClusterIP.

2. **PostgREST (API REST - Stateless):** 
   - API ligada dinamicamente à base de dados utilizando o DNS interno do Kubernetes.
   - Configurações de ambiente separadas em `ConfigMaps` (dados não sensíveis) e `Secrets` (credenciais).
   - Alta disponibilidade e resiliência garantidas via `livenessProbe` e `readinessProbe`.
   - Limites de recursos (`requests` e `limits` de CPU/Memória) estabelecidos para evitar sobrecarga no cluster.
   - Escalamento automático configurado via `HorizontalPodAutoscaler` (HPA).

---

## Tecnologias e Recursos Utilizados

- **Ambiente:** Fedora Linux, Kubernetes (K3s), `kubectl`.
- **Contentores:** PostgreSQL (v13), PostgREST (v12.0.2).
- **Recursos K8s Implementados:** 
  - `Namespace`, `Deployment`, `Service` (ClusterIP).
  - `PersistentVolumeClaim` (RWO).
  - `ConfigMap` e `Secret`.
  - `HorizontalPodAutoscaler` (HPA) integrado com o `metrics-server`.
  - `Probes` (Liveness e Readiness) e `Resources` (Requests/Limits).

---

## Estrutura de Diretórios

Os manifestos estão organizados de forma sequencial na pasta `evidencias/` e `k8s/`:

```text
├── evidencias
│   ├── images
│     ├── nivel1
│     ├── nivel2
│     ├── nivel3
│     ├── nivel4
│     ├── nivel5
│     ├── nivel6
│     └── nivel7
└── Evidencias.md
├── k8s/
│   ├── 00-namespace.yml
│   ├── 01-pod-teste.yaml
│   ├── 02-postgres-pv.yaml
│   ├── 02-postgres-pvc.yaml
│   ├── 03-postgres-service.yaml
│   ├── 04-postgres-deployment.yaml
│   ├── 05-postgres-config.yaml
│   ├── 06-postgrest-secret.yaml
│   ├── 07-app-deployment.yaml
│   ├── 07-postgrest-deployment.yaml
│   └── 08-postgrest-hpa.yaml
└── README.md
```

---

## Como Executar o Projeto

### Pré-requisitos
- Um cluster Kubernetes ativo (ex: K3s, Minikube ou Kind).
- `kubectl` instalado e configurado.
- `metrics-server` ativado no cluster (necessário para o HPA).

### Passo a Passo

**1. Clonar o repositório:**
```bash
git clone [https://github.com/DiegoQueiroz01/Desafio_II_Fund_Kubernetes.git](https://github.com/DiegoQueiroz01/Desafio_II_Fund_Kubernetes.git)
cd NOME-DO-REPOSITORIO
```

**2. Criar o Namespace:**
Antes de aplicar os restantes recursos, crie o namespace isolado do projeto:
```bash
kubectl apply -f k8s/01-namespace.yaml
```

**3. Configurar as Credenciais Seguras (Secret):**
Para evitar *hardcoding* de palavras-passe no repositório, crie o `Secret` diretamente no cluster:
```bash
kubectl create secret generic postgres-secret \
  --from-literal=uri='postgres://appuser:password@postgres-service:5432/appdb' \
  -n segundodesafio
```

**4. Aplicar o resto da infraestrutura:**
```bash
kubectl apply -f k8s/
```

**5. Verificar o estado dos recursos:**
```bash
kubectl get all -n segundodesafio
```

---

## Como Testar as Funcionalidades

### 1. Acesso à API a partir do exterior do cluster
Como o Service é do tipo `ClusterIP`, utilize o `port-forward` para expor a API localmente:
```bash
kubectl port-forward svc/postgrest-service 3000:3000 -n segundodesafio
```
Aceda no navegador ou via cURL: `http://localhost:3000/`

### 2. Teste de Persistência (PVC)
1. Insira um dado na base de dados através da API.
2. Elimine o Pod do PostgreSQL simulando uma falha:
   ```bash
   kubectl delete pod -l app=postgres -n segundodesafio
   ```
3. Aguarde que o Kubernetes recrie o Pod.
4. Faça um novo pedido à API e confirme que os dados continuam intactos.

### 3. Teste de Auto-Scaling (HPA)
Gere um pico de pedidos utilizando um Pod temporário com o `busybox` para colocar a CPU da API sob stress:
```bash
kubectl run load-gen --image=busybox -n segundodesafio -- /bin/sh -c "while true; do wget -q -O- http://postgrest-service:3000/ > /dev/null; done"
```

Acompanhe o escalamento das réplicas em tempo real:
```bash
kubectl get hpa postgrest-hpa -n segundodesafio -w
```
*(Verá o HPA escalar as réplicas automaticamente de 1 até 5, conforme o uso da CPU ultrapassa a meta de 50%).*

Para interromper o teste e observar o *scale-down*:
```bash
kubectl delete pod load-gen -n segundodesafio
```

---

## Conclusão e Aprendizagens
Este projeto consolidou conceitos avançados de orquestração de contentores, incluindo:
- Desacoplamento de configurações e segurança de credenciais.
- Gestão de estado em aplicações K8s (Stateful vs Stateless).
- Estratégias de auto-recuperação (Self-healing com Probes).
- Gestão de capacidade elástica baseada no consumo real de recursos (HPA).