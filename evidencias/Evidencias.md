# Desafio: Fundamentos de Kubernetes na Prática

#### Objetivo: Demonstrar conhecimento prático nos fundamentos de Kubernetes.

## Nível 1 - Namespace e Primeiro Contato:

![alt text](images/nivel1/print-1.1.png)
Print 1.1: Inspeção de eventos do Pod de teste inicializado no namespace dedicado.

![alt text](images/nivel1/print-1.2.png)
![alt text](images/nivel1/print-1.3.png)
Print 1.2 e 1.3: Terminal mostrando o resultado de "kubectl delete pod...", seguido de "kubectl get pods...", mostrando que não há recursos.

#### Reflexão: Ao deletar o Pod avulso, ele não irá retornar sozinho, pois não existe um controlar, como Deployment, gerenciando seu estado. Isso demonstrar como porque criamos raramente Pods em produção.