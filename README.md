# ipsense-EKS

## Depuração de pods e contêineres

# Logs de contêineres

A maneira de depurar um log de contêiner é por meio de um mecanismo de log. No Kubernetes, 
cada contêiner geralmente grava na saída padrão (stdout) e nos fluxos de erro padrão (stderr), 
a menos que algo diferente da abordagem de registro padrão tenha sido configurado, por exemplo, salvar em arquivo .log . 

Os registros do aplicativo podem ser recuperados usando:

kubectl -n logs <pod-name> 
kubectl -n logs <pod-name> --container <container-name>.


## Interagindo com pods em execução

kubectl logs my-pod                                 # dump pod logs (stdout)

kubectl logs -l name=myLabel                        # dump pod logs, with label name=myLabel (stdout)

kubectl logs my-pod --previous                      # dump pod logs (stdout) for a previous instantiation of a container

kubectl logs my-pod -c my-container                 # dump pod container logs (stdout, multi-container case)

kubectl logs -l name=myLabel -c my-container        # dump pod logs, with label name=myLabel (stdout)

kubectl logs my-pod -c my-container --previous      # dump pod container logs (stdout, multi-container case) for a previous instantiation of a container

kubectl logs -f my-pod                              # stream pod logs (stdout)

kubectl logs -f my-pod -c my-container              # stream pod container logs (stdout, multi-container case)

kubectl logs -f -l name=myLabel --all-containers    # stream all pods logs with label name=myLabel (stdout)

kubectl run -i --tty busybox --image=busybox -- sh  # Run pod as interactive shell

kubectl run nginx --image=nginx -n 

mynamespace                                         # Run pod nginx in a specific namespace

kubectl run nginx --image=nginx                     # Run pod nginx and write its spec into a file called pod.yaml
--dry-run=client -o yaml > pod.yaml

kubectl attach my-pod -i                            # Attach to Running Container

kubectl port-forward my-pod 5000:6000               # Listen on port 5000 on the local machine and forward to port 6000 on my-pod

kubectl exec my-pod -- ls /                         # Run command in existing pod (1 container case)

kubectl exec --stdin --tty my-pod -- /bin/sh        # Interactive shell access to a running pod (1 container case) 

kubectl exec my-pod -c my-container -- ls /         # Run command in existing pod (multi-container case)

kubectl top pod POD_NAME --containers               # Show metrics for a given pod and its containers

kubectl top pod POD_NAME --sort-by=cpu              # Show metrics for a given pod and sort it by 'cpu' or 'memory'


## Interagindo com nós e cluster 

kubectl cordon my-node                                                # Mark my-node as unschedulable

kubectl drain my-node                                                 # Drain my-node in preparation for 
maintenance

kubectl uncordon my-node                                              # Mark my-node as schedulable

kubectl top node my-node                                              # Show metrics for a given node

kubectl cluster-info                                                  # Display addresses of the master and services

kubectl cluster-info dump                                             # Dump current cluster state to stdout

kubectl cluster-info dump --output-directory=/path/to/cluster-state   # Dump current cluster state to /path/to/cluster-state


# If a taint with that key and effect already exists, its value is replaced as specified.
kubectl taint nodes foo dedicated=special-user:NoSchedule

## Depurar pods em execução
Esta página explica como depurar pods em execução (ou com falha) em um nó.

Antes de você começar Seu Podjá deve estar programado e funcionando. Se seu pod ainda não estiver em execução, comece com Solução de problemas de aplicativos . Para algumas das etapas de depuração avançada, você precisa saber em qual nó o pod está sendo executado e ter acesso ao shell para executar comandos nesse nó. Você não precisa desse acesso para executar as etapas de depuração padrão que usa kubectl.

# Examinando pod logs

Primeiro, observe os registros do contêiner afetado:

kubectl logs ${POD_NAME} ${CONTAINER_NAME}
Se seu contêiner já travou, você pode acessar o registro de falha do contêiner anterior com:

kubectl logs --previous ${POD_NAME} ${CONTAINER_NAME}
Depurando com o Container Exec
Se o imagem do recipienteinclui utilitários de depuração, como é o caso de imagens criadas a partir de imagens de base dos sistemas operacionais Linux e Windows, você pode executar comandos dentro de um contêiner específico com kubectl exec:

kubectl exec ${POD_NAME} -c ${CONTAINER_NAME} -- ${CMD} ${ARG1} ${ARG2} ... ${ARGN}
Nota: -c ${CONTAINER_NAME} é opcional. Você pode omiti-lo para pods que contêm apenas um único contêiner.
Por exemplo, para ver os registros de um pod do Cassandra em execução, você pode executar

kubectl exec cassandra -- cat /var/log/cassandra/system.log
Você pode executar um shell conectado ao seu terminal usando os argumentos -ie -tpara kubectl exec, por exemplo:

kubectl exec -it cassandra -- sh
Para obter mais detalhes, consulte Obter um shell para um contêiner em execução .

Depuração com um contêiner de depuração efêmero
ESTADO DE RECURSO: Kubernetes v1.18 [alpha]
Recipientes efêmeros são úteis para solução de problemas interativos quando kubectl execinsuficiente porque um contêiner travou ou uma imagem de contêiner não inclui utilitários de depuração, como com imagens sem distração . kubectltem um comando alfa que pode criar contêineres efêmeros para depuração começando com a versão v1.18.

Exemplo de depuração usando contêineres efêmeros
Observação: os exemplos nesta seção exigem o EphemeralContainers gate de recurso habilitado em seu cluster e kubectlversão v1.18 ou posterior.
Você pode usar o kubectl debugcomando para adicionar contêineres efêmeros a um pod em execução. Primeiro, crie um pod para o exemplo:

kubectl run ephemeral-demo --image=k8s.gcr.io/pause:3.1 --restart=Never
Os exemplos nesta seção usam a pauseimagem do contêiner porque ela não contém utilitários de depuração do espaço do usuário, mas esse método funciona com todas as imagens do contêiner.

Se você tentar usar kubectl execpara criar um shell, verá um erro porque não há shell nesta imagem de contêiner.

kubectl exec -it ephemeral-demo -- sh
OCI runtime exec failed: exec failed: container_linux.go:346: starting container process caused "exec: \"sh\": executable file not found in $PATH": unknown
Em vez disso, você pode adicionar um contêiner de depuração usando kubectl debug. Se você especificar o argumento -i/ --interactive, kubectlserá automaticamente anexado ao console do Recipiente Ephemeral.

kubectl debug -it ephemeral-demo --image=busybox --target=ephemeral-demo
Defaulting debug container name to debugger-8xzrl.
If you don't see a command prompt, try pressing enter.
/ #
Este comando adiciona um novo contêiner de busybox e anexa a ele. O --target parâmetro tem como alvo o namespace do processo de outro contêiner. É necessário aqui porque kubectl runnão ativa o compartilhamento de namespace de processo no pod que ele cria.

Nota: O --targetparâmetro deve ser compatível com oContainer Runtime. Quando não suportado, o Ephemeral Container pode não ser iniciado ou pode ser iniciado com um namespace de processo isolado.
Você pode visualizar o estado do contêiner efêmero recém-criado usando kubectl describe:

kubectl describe pod ephemeral-demo
...
Ephemeral Containers:
  debugger-8xzrl:
    Container ID:   docker://b888f9adfd15bd5739fefaa39e1df4dd3c617b9902082b1cfdc29c4028ffb2eb
    Image:          busybox
    Image ID:       docker-pullable://busybox@sha256:1828edd60c5efd34b2bf5dd3282ec0cc04d47b2ff9caa0b6d4f07a21d1c08084
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 12 Feb 2020 14:25:42 +0100
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:         <none>
...
Use kubectl deletepara remover o pod quando terminar:

kubectl delete pod ephemeral-demo
Depuração usando uma cópia do pod
Às vezes, as opções de configuração do pod dificultam a solução de problemas em certas situações. Por exemplo, você não pode executar kubectl execpara solucionar problemas do seu contêiner se a imagem do contêiner não incluir um shell ou se seu aplicativo travar na inicialização. Nessas situações, você pode usar kubectl debugpara criar uma cópia do pod com valores de configuração alterados para ajudar na depuração.

Copiar um pod ao adicionar um novo contêiner
Adicionar um novo contêiner pode ser útil quando seu aplicativo está em execução, mas não está se comportando como esperado e você gostaria de adicionar outros utilitários de solução de problemas ao pod.

Por exemplo, talvez as imagens de contêiner do seu aplicativo sejam criadas, busybox mas você precise de utilitários de depuração não incluídos em busybox. Você pode simular este cenário usando kubectl run:

kubectl run myapp --image=busybox --restart=Never -- sleep 1d
Execute este comando para criar uma cópia do myappnamed myapp-debugque adiciona um novo contêiner do Ubuntu para depuração:

kubectl debug myapp -it --image=ubuntu --share-processes --copy-to=myapp-debug
Defaulting debug container name to debugger-w7xmf.
If you don't see a command prompt, try pressing enter.
root@myapp-debug:/#
Observação:
kubectl debuggera automaticamente um nome de contêiner se você não escolher um usando o --containersinalizador.
O -isinalizador faz kubectl debugcom que seja anexado ao novo contêiner por padrão. Você pode evitar isso especificando --attach=false. Se sua sessão for desconectada, você pode reconectá-la usando kubectl attach.
O --share-processespermite que os contentores desta vagem para ver os processos dos outros recipientes no vagem. Para obter mais informações sobre como isso funciona, consulte Compartilhar namespace de processo entre contêineres em um pod .
Não se esqueça de limpar o pod de depuração quando terminar:

kubectl delete pod myapp myapp-debug
Copiar um pod enquanto muda seu comando
Às vezes, é útil alterar o comando de um contêiner, por exemplo, para adicionar um sinalizador de depuração ou porque o aplicativo está travando.

Para simular um aplicativo com falha, use kubectl runpara criar um contêiner que sai imediatamente:

kubectl run --image=busybox myapp -- false
Você pode ver kubectl describe pod myappque este contêiner está travando:

Containers:
  myapp:
    Image:         busybox
    ...
    Args:
      false
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
Você pode usar kubectl debugpara criar uma cópia deste pod com o comando alterado para um shell interativo:

kubectl debug myapp -it --copy-to=myapp-debug --container=myapp -- sh
If you don't see a command prompt, try pressing enter.
/ #
Agora você tem um shell interativo que pode usar para realizar tarefas como verificar caminhos do sistema de arquivos ou executar o comando do contêiner manualmente.

Observação:
Para alterar o comando de um contêiner específico, você deve especificar seu nome usando --containerou kubectl debug, em vez disso, criará um novo contêiner para executar o comando especificado.
O -isinalizador faz kubectl debugcom que seja anexado ao contêiner por padrão. Você pode evitar isso especificando --attach=false. Se sua sessão for desconectada, você pode reconectá-la usando kubectl attach.
Não se esqueça de limpar o pod de depuração quando terminar:

kubectl delete pod myapp myapp-debug
Copiar um pod ao alterar as imagens do contêiner
Em algumas situações, você pode querer alterar um pod com comportamento incorreto de suas imagens de contêiner de produção normal para uma imagem contendo uma compilação de depuração ou utilitários adicionais.

Por exemplo, crie um pod usando kubectl run:

kubectl run myapp --image=busybox --restart=Never -- sleep 1d
Agora use kubectl debugpara fazer uma cópia e alterar sua imagem de contêiner para ubuntu:

kubectl debug myapp --copy-to=myapp-debug --set-image=*=ubuntu
A sintaxe de --set-imageusa a mesma container_name=imagesintaxe de kubectl set image. *=ubuntusignifica alterar a imagem de todos os contêineres para ubuntu.

Não se esqueça de limpar o pod de depuração quando terminar:

kubectl delete pod myapp myapp-debug
Depuração por meio de um shell no nó
Se nenhuma dessas abordagens funcionar, você pode encontrar o nó no qual o pod está sendo executado e criar um pod privilegiado em execução nos namespaces do host. Para criar um shell interativo em um nó usando kubectl debug, execute:

kubectl debug node/mynode -it --image=ubuntu
Creating debugging pod node-debugger-mynode-pdx84 with container debugger on node mynode.
If you don't see a command prompt, try pressing enter.
root@ek8s:/#
Ao criar uma sessão de depuração em um nó, lembre-se de que:

kubectl debug gera automaticamente o nome do novo pod com base no nome do nó.
O contêiner é executado nos namespaces IPC, Rede e PID do host.
O sistema de arquivos raiz do Node será montado em /host.
Não se esqueça de limpar o pod de depuração quando terminar:

kubectl delete pod node-debugger-mynode-pdx84

