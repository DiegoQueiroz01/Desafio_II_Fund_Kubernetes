# Desafio: Fundamentos de Kubernetes na Prática

#### Objetivo: Demonstrar conhecimento prático nos fundamentos de Kubernetes.

## Nível 1 - Namespace e Primeiro Contato:

![alt text](images/nivel1/print-1.1.png)
Print 1.1: Inspeção de eventos do Pod de teste inicializado no namespace dedicado.

![alt text](images/nivel1/print-1.2.png)
![alt text](images/nivel1/print-1.3.png)
Print 1.2 e 1.3: Terminal mostrando o resultado de "kubectl delete pod...", seguido de "kubectl get pods...", mostrando que não há recursos.

#### Reflexão: Ao deletar o Pod avulso, ele não irá retornar sozinho, pois não existe um controlador, como Deployment, gerenciando seu estado. Isso demonstrar como porque criamos raramente Pods em produção.

## Nível 2 - Banco de Dados com Persistência

![alt text](images/nivel2/print%20-%202.0.png)
Print 2.0: Terminal mostrando a saída do comando "kubectl get pv,pvc,pods -n segundodesafio" e "kubectl get pods -n segundodesafio", comprovando a criação do espaço em memória, de 1GB.

#### Reflexão: A principal diferença é a persistência e o ciclo de vida. Um volume do tipo emptyDir é temporário e está atrelado ao ciclo de vida do Pod; se o Pod for deletado ou falhar, os dados são apagados permanentemente. Já o PVC solicita armazenamento físico real. Se o Pod for deletado, os dados permanecem intactos no disco físico e são reconectados automaticamente quando o Pod for recriado.

## Nível 3 - ConfigMap e Secrets
![alt text](images/nivel3/print-3.0.png)
Print 3.0: Validação da criação do configmap/postgres-config, secret/postgres-secret e reinicialização limpa do Deployment do PostgreSQL utilizando referências seguras (configMapKeyRef e secretKeyRef).

#### Boas práticas:
* ConfigMap: Armazena dados de configuração não sensíveis em texto claro (POSTGRES_DB, POSTGRES_USER), permitindo reutilização e alteração sem rebuild de imagem.
* Secret: Armazena credenciais confidenciais codificadas em Base64 (c2VuaGExMjM= para a senha do postgres), evitando expor senhas em texto plano no manifesto do Deployment.

#### Reflexão: É apenas codificação, não criptografia. Qualquer pessoa com acesso ao kubectl get secret descodifica a string instantaneamente. Para a segurança real, isso significa que o Secret padrão do Kubernetes por si só não protege contra vazamentos de privilégio no cluster ou acesso indevido ao etcd em claro.

