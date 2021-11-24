# Velero

Velero é uma ferramenta de código aberto para fazer backup e restaurar com segurança, realizar recuperação de desastres e migrar recursos de cluster do Kubernetes e volumes persistentes.

## Instalando o Velero

No passo anterior provisionamos dois clusters, um com o `kOps` e outro com o `eksctl`, neste modulo aprendere-mos como vamos realizar a instalação do `velero` para migrar um cluster Kubernetes do auto-gerenciado para o gerenciado.

## Instalando Velero no auto-gerenciado

Nesta etapa do workshop faremos a instalação do Velero no cluster provisionado via kOps.

### Criando o Bucket para salvar os backups

```sh
export BUCKET_NAME=velero-backup-workshop-bucket

aws s3 mb s3://${BUCKET_NAME} --region us-east-1
```

- Criando a policy para associar a Role dos Nodes.

```sh
cat > velero-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeVolumes",
                "ec2:DescribeSnapshots",
                "ec2:CreateTags",
                "ec2:CreateVolume",
                "ec2:CreateSnapshot",
                "ec2:DeleteSnapshot"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET_NAME}/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET_NAME}"
            ]
        }
    ]
}
EOF
```

- Criando o velero user e adicionando a policy

```sh
aws iam create-user --user-name velero

aws iam put-user-policy \
  --user-name velero \
  --policy-name velero \
  --policy-document file://velero-policy.json
```

- Criando a access key para o usuario

```sh
aws iam create-access-key --user-name velero
```

- O resultado será parecido com o seguinte

```json
{
  "AccessKey": {
        "UserName": "velero",
        "Status": "Active",
        "CreateDate": "2017-07-31T22:24:41.576Z",
        "SecretAccessKey": <AWS_SECRET_ACCESS_KEY>,
        "AccessKeyId": <AWS_ACCESS_KEY_ID>
  }
}
```

- Criando o arquivo de credenciais para o velero `credentials-velero`

```sh
[default]
aws_access_key_id=<AWS_ACCESS_KEY_ID>
aws_secret_access_key=<AWS_SECRET_ACCESS_KEY>
```

### Realizando a instalação via velero cli

- Instale o velero cli utilizando o seguinte [link](https://velero.io/docs/v1.7/basic-install/)

- Instalando o Velero

```sh
export REGION=us-east-1

velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.3.0 \
    --bucket $BUCKET_NAME \
    --backup-location-config region=$REGION \
    --snapshot-location-config region=$REGION \
    --secret-file ./credentials-velero \
    --use-restic
```

### Deploy Wordpress

O Wordpress possui volumes, ou seja, tem persistência de dados em um PV, que por sua vez é um Amazon EBS, para que consigamos realizar a migração entre clusters e o Backup do PV é necessário utilizar o [restic](https://github.com/restic/restic)

O Velero oferece suporte para backup e restauração de volumes do Kubernetes usando uma ferramenta de backup de código aberto gratuita chamada restic.

> Este suporte é considerado qualidade beta. Consulte a lista de limitações para entender se ela se ajusta ao seu caso de uso.

O Velero permite que você tire snapshots de volumes persistentes como parte de seus backups se você estiver usando um dos provedores de nuvem com suporte de ofertas de armazenamento em bloco (Amazon EBS Volumes, Azure Managed Disks, Google Persistent Disks). Ele também fornece um modelo de plug-in que permite que qualquer pessoa implemente objetos adicionais e back-ends de armazenamento em bloco, fora do repositório principal do Velero.

- Vamos realizar o deploy de um wordpress para testarmos o backup

```sh
kubectl create ns wordpress
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-release bitnami/wordpress -n wordpress
```

### Criando um backup com o Velero

- Realizando o backup do nosso ambiente wordpress

```sh
velero backup create wordpress-backup --include-namespaces wordpress

velero backup describe wordpress-backup

velero backup describe wordpress-backup --details

velero backup get
```

## Instalando Velero no EKS

Nós utilizaremos o mesmo usuário e cli utilizado anteriormente para poupar nosso tempo, então basta definirmos o contexto do outro cluster que faremos a instalação e migração do Wordpress.

- Realizando o update do kubeconfig adicionando o eks.

```sh
aws eks --region <region-code> update-kubeconfig --name <cluster_name>
```

- Instalando o velero utilizando o velero cli.

```sh
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.3.0 \
    --bucket $BUCKET_NAME \
    --backup-location-config region=$REGION \
    --snapshot-location-config region=$REGION \
    --secret-file ./credentials-velero \
    --use-restic
```

## Migrando a aplicação do cluster auto-gerenciado para o Amazon EKS

```sh
velero restore create --from-backup wordpress-backup --include-namespaces wordpress
```
