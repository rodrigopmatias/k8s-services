## Pacote de Serviços que uso no meu dia-a-dia como Developer

Ao longo do tempo passe a acumular algum serviços que uso no meu dia a dia como develop, serviços básicos como database, mensageria entre outros que me ajuda a desenvolver sem precisar de infraestrutura externa, são eles até o monento:

1. PostgreSQL
1. RabbitMQ
1. Redis
1. Docker Registry
1. PyPi (package registry for Python)

O que pode soar como extranho é a implementação do PyPi, este eu uso para desenvolvimento de projetos pessoais que tenho pacotes privados meus.

Para seguirmos com a implementação dos serviços sugiro a seguinte ordem:

1. [Implementando o Regitry](#implementado-o-registry)
1. [Implementando o PyPI Server](#implementando-o-pypi-server)
1. [Implementando os fixes](#implementado-os-fixes)
1. [Implementando o Redis](#implementado-redis)
1. [Implementando o PostgreSQL](#implementando-postgresql)
1. [Implementnando o RabbitMQ](#implementando-o-rabbitmq)

Necessáriamente os 3 primeiros são importantes para o sucesso, após eles os demais podem ser inclusive omitidos ou substituidos pelos de sua preferencia.

## Implementado o Registry

Você irá precisar de um docker para poder rodar o comando `htpasswd` você pode procurar outra alternativa, mas para gerar o nosso arquivo de autenticação iremos rodar o seguind comando:

```shell
docker run -it --rm httpd:2-alpine htpasswd -nB pipeline
```
Tendo a seguinte saída:

```
Password:
Confirm Password: 
pipeline:hiden-value
```

Esta ultima linha corresponde a entrada que iremos adicionar no arquivo `password.txt` dentro do diretório registry.

Agora vamos criar o arquivo de secrets, onde iremos inserir o conteudo do password.txt em uma entrada que será usada pelo registry para buscar informações de usuários veja o comando:

```shell
kubectl create secret generic registry-secrets \
  --from-file password.txt=registry/password.txt \
  --dry-run=client \
  --output=yaml > registry/registry-secrets.yaml
```

Este comando irá gerar o arquivo `registry-secrets.yaml` dentro do diretório `registry` e agora pode aplicar o deploy com o seguinte comando:

```shell
kubectl apply -n infra-registry -f registry
```

Isto irá fazer com que todos os artefatos de configuração sejam instanciados no nosso kubernetes, podemos verificar com o seguinte comando:

```shell
kubectl get all -n infra-registry
```

Tendo a seguinte saída:

```
NAME             READY   STATUS    RESTARTS   AGE
pod/registry-0   1/1     Running   0          4m35s

NAME               TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/registry   ClusterIP   10.43.110.70   <none>        5000/TCP   16m

NAME                        READY   AGE
statefulset.apps/registry   1/1     4m35s
```

Quando precisarmos acessar imagens no registry iremos sempre precisar informar as credenciais de acesso para o mesmo, para isto iremos criar um arquivo que devemos deixar em um local seguro para isto, veja o comando:

```
kubectl create secret docker-registry registry-secrets \
  --docker-username my-user \
  --docker-password my-password \
  --docker-server registry.internal:5000 \
  --dry-run=client \
  --output=yaml > registry-secrets.yaml
```

Desta forma será criado um arquivo `registry-secrets.yaml` onde temos a configuração do docker para acessar o nosso registry.

Ainda como o nosso registry não possui um certificado válido, precsamos fazer uma configuração no `k3s` para que possamos utiliza-lo sem problema, primeiro iremos criar o arquivo `/etc/rancher/k3s/registries.yaml` com o seguinte conteúdo:

```yaml
mirrors:
  "registry.internal:5000":
    endpoint:
      - "http://registry.internal:5000"
```

Em seguida precisamos reiniciar o `k3s` use o comando `sudo systemctl restart k3s` neste momento o kubernetes já consegue acessar imagens que estejam armazenadas neste serviço.

## Implementando o pypi-server

Você irá precisar de um docker para poder rodar o comando `htpasswd` você pode procurar outra alternativa, mas para gerar o nosso arquivo de autenticação iremos rodar o seguind comando:

```shell
docker run -it --rm httpd:2-alpine htpasswd -nB pipeline
```

Tendo a seguinte saída:

```
Password:
Confirm Password: 
pipeline:hiden-value
```

Esta ultima linha corresponde a entrada que iremos adicionar no arquivo `password.txt` dentro do diretório registry.

Agora vamos criar o arquivo de secrets, onde iremos inserir o conteudo do password.txt em uma entrada que será usada pelo registry para buscar informações de usuários veja o comando:

```shell
kubectl create secret generic pypi-secrets \
  --from-file password.txt=pypi/password.txt \
  --dry-run=client \
  --output=yaml > pypi/pypi-secrets.yaml
```

Este comando irá gerar o arquivo `pypi-secrets.yaml` dentro do diretório `pypi` e agora pode aplicar o deploy com o seguinte comando:

```shell
kubectl apply -n infra-pypi -f pypi
```

Isto irá fazer com que todos os artefatos de configuração sejam instanciados no nosso kubernetes, podemos verificar com o seguinte comando:

```shell
kubectl get all -n infra-pypi 
```

Tendo a seguinte saída:

```
NAME                        READY   STATUS    RESTARTS   AGE
pod/pypi-79669cfd74-cbbvt   1/1     Running   0          116s

NAME           TYPE           CLUSTER-IP    EXTERNAL-IP       PORT(S)          AGE
service/pypi   LoadBalancer   10.43.25.49   192.168.101.210   8000:31480/TCP   116s

NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/pypi   1/1     1            1           116s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/pypi-79669cfd74   1         1         1       117s
```

## Implementado os fixes

Atualmente temos apenas um fix que irá criar entradas de DNS a nivel de host para que seja possivel o kubernetes interagir com algums serviços internos de forma mais simples.

Vamos começar aplicando o configmap pois teremos que realizar algumas adequações, verifique o `IP_VIP` que deve ser o `IP` da maquina virtual, e em seguida os serviços que serão incluidas as entradas de `DNS`, neste caso cada linha representa um serviço onde temos o seguinte formato:

```
namespace:servico:dns
```

Exemplo:

```
infra-registry:registry:registry.internal
```

Neste caso o serviço `registry` esta presente no namespace `infra-registry` e será registrado om o nome `registry.internal`.

## Implementado Redis

Assim como [Implementando PostgreSQL](#implementando-postgresql) iremos precisar do `registry` funcionando.

A implementação de Redis é uma simples implementação para ambiente de desenvolvimento, não contando com replicação nem com autenticação, mesmo que isto sejá facil de implementar, mas para implementar o Redis basta executar o seguinte comando:

```
kubectl apply -n data -f data/redis
```

Tendo a seguinte saída:

```
deployment.apps/redis created
service/redis created
```

Para ter acesso execute este comando:

```
kubectl get svc -n data redis
```

Tendo a seguinte saída:

```
NAME    TYPE           CLUSTER-IP    EXTERNAL-IP       PORT(S)          AGE
redis   LoadBalancer   10.43.91.59   192.168.101.210   6379:32326/TCP   29s
```

Sendo assim podemos acessar este redis da maquina de desenvolvimento com a seguinte URL `redis://192.168.101.210:6379/1`.

## Implementando PostgreSQL

Antes de realizarmos o deploy deste item, precisamos ter o `registry` funcionando uma vez que a imagem do postgres será buscada neste, veja [Implementando registry](#implementado-o-registry) iremos precisar do usuário e senha para podemos fazer o upload da imagem e depoiis gerar o arquivo registry-secrets.yaml que é necessário para conseguirmos acessar as imagens no nosso registry privado.

O usuário do postgres que será criado pode ser verificado no arquivo configmap, por padrão será o usuário `joe` você pode modificar para o valor que deseja, mas, isto precisa ser feito antes do primeiro deploy, a senha padrão é uma senha fraca, você também pode modificar este valor antes do primeiro deploy, caso tenha esquecido de fazer isto o postgreSQL conta com ferramentas que podem te auxiliar, para isto deve consultar documentação do PostgreSQL.

O deploy é bem simples, basta executar o seguinte comando:

```
kubectl apply -n data -f data/postgres
```

Tendo a seguinte saída:

```
configmap/postgres-configmap created
deployment.apps/postgres created
persistentvolumeclaim/postgres-pvc created
secret/postgres-secrets created
service/postgres created
```

Este serviço é executado como LoadBalancer logo ele possui um IP externo que pode ser verificado com o seguinte comando:

```
kubectl get svc -n data postgres
```

Tendo a seguinte saída:

```
NAME       TYPE           CLUSTER-IP    EXTERNAL-IP       PORT(S)          AGE
postgres   LoadBalancer   10.43.71.43   192.168.101.210   5432:30110/TCP   68s
```

No meu caso foi o IP `192.168.101.210` a porta `5432`.

## Implementando o RabbitMQ

Neste deploy temos configurados alguns plugins e persistencia de dados para que ao reiniciar o Pod não seja perdida configurações de topicos e queues nem mesmo de acesso.

Podemos implementar o serviço com o simples comando:

```shell
kubectl apply -n data -f data/rabbit
```

Tendo a seguinte saída:

```
configmap/rabbit-configmap created
deployment.apps/rabbit created
persistentvolumeclaim/rabbit-pvc created
secret/rabbit-secrets created
service/rabbit created
```

Lembando que o serviço será iniciao com um usuário `admin` que possue uma senha muito fraca, você pode alterar esta senha antes do deploy no arquivo `secrets.yaml` ou depois atraves do painel e administração que ficará disponível no endereço `http://ip-do-k8s:15672/`. Este IP também pode ser descober com o seguinte comando:

```shell
kubectl get svc -n data rabbit
```

Tendo a seguinte saída:

```
NAME     TYPE           CLUSTER-IP   EXTERNAL-IP       PORT(S)                                                          AGE
rabbit   LoadBalancer   10.43.5.57   192.168.101.210   5672:31243/TCP,15672:31194/TCP,15692:31943/TCP,61613:32328/TCP   2m46s
```

Devemos observar o valor do `EXTERNAL-IP` que no meu caso é `192.168.101.210` veja como será o seu e substitua.