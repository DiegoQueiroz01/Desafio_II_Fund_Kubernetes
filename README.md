# Desafio II - Fundamentos de Kubernetes

Implantacao local de uma API PostgREST integrada a PostgreSQL, com Namespace,
ConfigMap, Secret, PVC, probes, requests/limits e HPA.

## Arquitetura

- `postgres`: Deployment com uma replica e armazenamento persistente via PVC.
- `postgres-service`: Service `ClusterIP` usado pelo PostgREST via DNS interno.
- `postgrest`: API REST na porta 3000.
- `postgrest-service`: acesso interno e alvo do `port-forward`.
- `postgrest-hpa`: escala a API de 1 a 5 replicas conforme CPU.

Todos os recursos ficam no namespace `segundodesafio`.

## Pre-requisitos

- Cluster local ativo: K3s, Rancher Desktop, Docker Desktop, Minikube ou Kind.
- `kubectl` configurado para o cluster.
- Metrics Server instalado para o HPA.
- Uma StorageClass padrao no cluster. O PVC usa a StorageClass padrao, sem
  depender de `hostPath` ou de um caminho especifico da maquina.

Verifique antes de iniciar:

```bash
kubectl get nodes
kubectl get storageclass
kubectl top nodes
```

## Instalação reproduzível

O Secret real nao fica no Git. Gere-o no cluster usando uma senha local:

```bash
export POSTGRES_PASSWORD='troque-por-uma-senha-local'
kubectl apply -f k8s/00-namespace.yml
kubectl create secret generic postgres-secret \
  --namespace segundodesafio \
  --from-literal=POSTGRES_PASSWORD="$POSTGRES_PASSWORD" \
  --from-literal=uri="postgres://appuser:${POSTGRES_PASSWORD}@postgres-service:5432/appdb"
```

Aplique os recursos principais. O comando nao inclui o arquivo de exemplo do
Secret, que fica fora de `k8s/` justamente para nao substituir o Secret real.

```bash
kubectl apply -f k8s/02-postgres-pvc.yaml
kubectl apply -f k8s/03-postgres-service.yaml
kubectl apply -f k8s/05-postgres-config.yaml
kubectl apply -f k8s/06-postgres-init-config.yaml
kubectl apply -f k8s/04-postgres-deployment.yaml
kubectl apply -f k8s/07-postgrest-deployment.yaml
kubectl apply -f k8s/08-postgrest-hpa.yaml
kubectl rollout status deployment/postgres -n segundodesafio
kubectl rollout status deployment/postgrest -n segundodesafio
```

O script SQL em `06-postgres-init-config.yaml` cria a tabela `tarefas` apenas
na inicializacao de um banco novo. Em um PVC ja existente, ele nao e executado
novamente, conforme o comportamento da imagem oficial do PostgreSQL.

## Verificação

```bash
kubectl get all,pvc -n segundodesafio
kubectl get hpa -n segundodesafio
```

Em outro terminal, exponha a API localmente:

```bash
kubectl port-forward svc/postgrest-service 3000:3000 -n segundodesafio
```

Em um terceiro terminal, teste leitura e escrita:

```bash
curl http://127.0.0.1:3000/tarefas
curl -X POST http://127.0.0.1:3000/tarefas \
  -H 'Content-Type: application/json' \
  -H 'Prefer: return=representation' \
  --data '{"titulo":"dado persistente","concluido":false}'
```

## Teste de persistencia

Anote o resultado do POST, remova o Pod do banco e aguarde a recriacao:

```bash
kubectl delete pod -l app=postgres -n segundodesafio
kubectl wait --for=condition=ready pod -l app=postgres \
  -n segundodesafio --timeout=180s
curl http://127.0.0.1:3000/tarefas
```

O registro criado antes da remocao deve continuar disponivel. O Deployment
recria o Pod e o PVC remonta os dados persistidos.

## Teste do HPA

Gere carga dentro do cluster:

```bash
kubectl run load-gen --image=busybox:1.36 -n segundodesafio --restart=Never \
  -- /bin/sh -c 'while true; do wget -q -O- http://postgrest-service:3000/tarefas >/dev/null; done'
kubectl get hpa postgrest-hpa -n segundodesafio -w
```

Finalize o gerador quando terminar:

```bash
kubectl delete pod load-gen -n segundodesafio
```

## Pod de teste opcional

O arquivo `k8s/01-pod-teste.yaml` existe para o Nivel 1 do desafio e nao faz
parte da instalacao principal. Para usa-lo:

```bash
kubectl apply -f k8s/01-pod-teste.yaml
kubectl describe pod nginx-avulso -n segundodesafio
kubectl logs nginx-avulso -n segundodesafio
kubectl delete pod nginx-avulso -n segundodesafio
```

## Migração de uma instalação antiga

Versoes anteriores usavam um PV estatico com `hostPath` e um PVC com
`storageClassName: ""`. Como `storageClassName` e imutavel em PVC vinculado,
uma instalacao antiga precisa ser recriada para usar a StorageClass padrao.
Faca backup antes:

```bash
kubectl exec -n segundodesafio deployment/postgres -- \
  pg_dump -U appuser -d appdb > backup.sql
```

Depois, em um ambiente de laboratorio, remova o namespace antigo e execute a
instalacao reproduzivel novamente. Restaure o backup se necessario:

```bash
kubectl delete namespace segundodesafio
kubectl delete pv postgres-pv --ignore-not-found
```

O template de Secret esta em
`examples/postgres-secret.example.yaml`; ele documenta o formato, mas nao deve
ser aplicado com `CHANGE_ME` em um ambiente real.

## Evidencias

As capturas do desafio estao em [evidencias/Evidencias.md](evidencias/Evidencias.md),
organizadas por nivel.

## Limpeza

```bash
kubectl delete namespace segundodesafio
```
