# Realizando deploy do seu cluster com kOps

Gostamos de pensar nisso como `kubectl` para clusters.

O `kops` não apenas ajudará você a criar, destruir, atualizar e manter o cluster Kubernetes de nível de produção e altamente disponível, mas também fornecerá a infraestrutura de nuvem necessária.

AWS (Amazon Web Services) é oficialmente compatível com DigitalOcean, GCE e OpenStack em suporte beta e Azure e AliCloud em alfa.

## Realizando a instalação do kOps

Realize o download do `kops` utilizando o seguinte [link](https://kubernetes.io/docs/setup/production-environment/tools/kops/)

## Criando bucket para realizar o store do state do cluster

O kops permite que você gerencie seus clusters mesmo após a instalação. Para fazer isso, ele deve manter o controle dos clusters que você criou, junto com sua configuração, as chaves que estão usando, etc. Essas informações são armazenadas em um bucket S3. As permissões S3 são usadas para controlar o acesso ao bucket.

No nosso exemplo escolhemos `k8s.local` como nossa hosted zone, então para o state do nosso cluster vamos escolher algo como `cluster.m2m.k8s.local`.

> se você estiver usando o Kops 1.6.2 ou posterior, a configuração do DNS é opcional. Em vez disso, um *gossip-based* cluster pode ser facilmente criado. O único requisito para acionar isso é que o nome do cluster termine com `.k8s.local.` Dessa maneira para acessar o cluster utilizaremos o endpoint do NLB que será criado pelo kOps.

- Crie o bucket.

```sh
aws s3 mb s3://cluster.workshop.k8s.local --region us-east-1
```

- Exportaremos o nome do bukect em uma variavel de ambiente para o `kops` saber onde ira salvar o state.

```sh
export KOPS_STATE_STORE=s3://cluster.workshop.k8s.local
```

## Criando a configuração do cluster

- Atualize o nome do bucket no manifesto.

```sh
sed -i s/__BUCKET_NAME__/${KOPS_STATE_STORE}/g ./kops-cluster-manifest.yaml
```

- Execute o comando `kops create -f` para criar a configuração do cluster.

```sh
kops create -f ./kops-cluster-manifest.yaml
```

> Nesse caso nós criaremos um cluster com 3 master nodes e um instance group com min 1 instância e máximo 10

> Ele apenas criará o state e a definição do que será criado

## Criando SSH key para autenticação

Chave utilizada para autenticação dos Nodes no cluster via SSH

Se você não tiver uma chave SSH crie uma acessando esse [link](https://docs.aws.amazon.com/en_us/cloudhsm/classic/userguide/generate_ssh_key.html#generate_ssh_key_linux)

- Importe a chave SSH publica para o kOps

```sh
kops create secret --name kops-m2m-cluster.k8s.local sshpublickey admin -i PATH-FOR-YOU-KEY
```

## Criando o cluster

- Execute o seguinte comando.

```sh
kops update cluster kops-m2m-cluster.k8s.local --yes
```

- Execute o comando de validação do kOps

```sh
kops validate cluster --wait 10m
```

## Verificando se o cluster foi criado corretamente

- Exporte a kubeconfig utilizando o Kops.

```sh
kops export kubecfg --admin
```

- Realize o `get nodes`

```sh
kubectl get nodes
```

```string
NAME                            STATUS   ROLES                  AGE   VERSION
ip-172-20-34-162.ec2.internal   Ready    control-plane,master   62m   v1.20.11
ip-172-20-44-116.ec2.internal   Ready    node                   60m   v1.20.11
ip-172-20-56-235.ec2.internal   Ready    control-plane,master   61m   v1.20.11
ip-172-20-67-23.ec2.internal    Ready    control-plane,master   61m   v1.20.11
```
