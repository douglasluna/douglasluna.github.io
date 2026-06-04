---
title: Use a Postura de Segurança do GKE para Detectar Problemas
date: 2023-08-23
lang: pt-BR
categories: [google-cloud, kubernetes]
tags: [google-cloud, gke, kubernetes, security, containers]
description: Como habilitar a Postura de Segurança do GKE, criar um workload de teste, encontrar problemas de configuração e acompanhar a remediação pelo Cloud Logging.
---

# Use a Postura de Segurança do GKE para Detectar Problemas

> Este texto foi migrado do meu blog anterior. Ele foi escrito originalmente em 2023, então confirme versões, preços e disponibilidade na documentação oficial antes de aplicar em produção.

## Introdução

O Google Cloud disponibilizou o painel de Postura de Segurança do GKE, ou GKE Security Posture, como uma ferramenta nativa para simplificar o gerenciamento de segurança e melhorar a proteção de clusters do Google Kubernetes Engine.

Esse painel ajuda a detectar configurações incorretas em clusters e cargas de trabalho do GKE, além de verificar vulnerabilidades em imagens de contêiner. Com isso, ele oferece recomendações acionáveis para fortalecer a postura de segurança e manter as aplicações mais protegidas.

## Como funciona

Para utilizar o painel de Postura de Segurança, basta habilitar um ou mais recursos compatíveis nos clusters que executam versões com suporte.

Os recursos são classificados em dois tipos principais.

### Postura de segurança do Kubernetes

Esse grupo inclui:

- Verificação da configuração da carga de trabalho: verifica automaticamente a configuração de workloads Kubernetes em execução nos clusters, como Pods, Jobs e StatefulSets.
- Exibição do boletim de segurança acionável: quando uma vulnerabilidade é descoberta no GKE, o Google corrige a vulnerabilidade e publica um boletim de segurança relacionado.

Esse recurso fica disponível em clusters GKE Autopilot ou Standard na versão 1.21 ou posterior. Em novos clusters executando a versão 1.27 ou posterior, ele é habilitado automaticamente.

### Scan de vulnerabilidade de contêiner

Esse recurso verifica automaticamente as imagens de contêiner usadas pelas cargas de trabalho em execução para encontrar vulnerabilidades conhecidas e, quando possível, sugerir estratégias de mitigação.

Na época em que este post foi escrito, existiam duas classes:

- Standard, ativada com a flag `standard`.
- Advanced vulnerability insights, ativada com a flag `enterprise`, ainda em Preview naquele momento.

Para a classe Standard, o recurso estava disponível no GKE Autopilot a partir da versão `1.23.5-gke.700` e era habilitado automaticamente em novos clusters Autopilot na versão 1.27 ou posterior. No GKE Standard, também estava disponível a partir da versão `1.23.5-gke.700`, mas não era ativado automaticamente.

Para a classe Enterprise, o cluster precisava estar na versão 1.27 ou superior, e o recurso vinha desabilitado por padrão nos modos Autopilot e Standard.

Depois de habilitado, o GKE verifica os clusters e as cargas de trabalho. O painel de Postura de Segurança no console do Google Cloud exibe os resultados e fornece recomendações acionáveis quando aplicável. O GKE também envia logs ao Cloud Logging para auditoria e rastreabilidade.

## Preço

Na época da publicação original, os recursos de verificação e o painel no console do Google Cloud eram oferecidos sem custo adicional. As informações adicionadas ao Cloud Logging seguiam os preços do Cloud Logging.

Como o scan de vulnerabilidade de pacotes de linguagem estava em Preview quando escrevi este texto, ele também não tinha custo adicional naquele momento.

## Pré-requisitos

Para ativar as verificações, precisamos habilitar algumas APIs no projeto:

- Google Kubernetes Engine API.
- Container Security API.
- Artifact Analysis API, apenas se você quiser usar o insight avançado de vulnerabilidades.

Abra o Cloud Shell e execute:

```bash
gcloud services enable container.googleapis.com containersecurity.googleapis.com containeranalysis.googleapis.com
```

Depois, valide se as APIs foram ativadas com sucesso:

```bash
gcloud services list | grep -E "container.googleapis.com|containersecurity.googleapis.com|containeranalysis.googleapis.com"
```

O output esperado deve ser parecido com:

```text
NAME: container.googleapis.com
NAME: containeranalysis.googleapis.com
NAME: containersecurity.googleapis.com
```

Podemos ativar a verificação de configuração de workload e o scan de vulnerabilidade de contêiner em novos clusters ou clusters já em execução, usando a CLI do Google Cloud ou o console do Google Cloud.

## Ativando a verificação da configuração da carga de trabalho

Vamos ativar manualmente esse recurso usando a CLI `gcloud`.

Vale lembrar: a verificação de postura de segurança do Kubernetes é habilitada por padrão nos novos clusters Autopilot e Standard executando a versão 1.27 ou posterior.

### Criando um novo cluster

Vamos criar um novo cluster GKE em modo Autopilot com a verificação de configuração de workload e o scan de vulnerabilidade de contêiner ativos:

```bash
gcloud container clusters create-auto NOME_DO_CLUSTER \
  --region=REGIAO_DO_COMPUTE \
  --security-posture=standard
```

Substitua:

- `NOME_DO_CLUSTER`: o nome desejado para o novo cluster.
- `REGIAO_DO_COMPUTE`: a região do Compute Engine do cluster.

Para clusters Standard zonais, utilize `--zone=ZONA_DO_COMPUTE`. Se você não especificar uma rede, a rede default do projeto será utilizada. Para definir uma rede, adicione `--network=O_NOME_DA_SUA_REDE`; para definir uma subnet, use `--subnetwork=O_NOME_DA_SUA_SUBNET`.

No meu caso, o comando ficou assim:

```bash
gcloud container clusters create-auto gkesp-cluster \
  --region=us-south1 \
  --security-posture=standard
```

> A criação do cluster deve demorar alguns minutos. No meu caso, levou cerca de 10 minutos.

Após a criação, tive um output parecido com:

```text
kubeconfig entry generated for gkesp-cluster.
NAME: gkesp-cluster
LOCATION: us-south1
MASTER_VERSION: 1.27.3-gke.100
MASTER_IP: 34.174.116.159
MACHINE_TYPE: e2-medium
NODE_VERSION: 1.27.3-gke.100
NUM_NODES: 3
STATUS: RUNNING
```

Para confirmar que o `kubectl` consegue acessar o cluster, execute:

```bash
kubectl get nodes
```

O output deve ser parecido com:

```text
NAME                                           STATUS   ROLES    AGE   VERSION
gk3-gkesp-cluster-default-pool-4e549890-l72s   Ready    <none>   10m   v1.27.3-gke.100
```

### Ativando em um cluster existente

Também é possível ativar a verificação de configuração de workload e o scan de vulnerabilidade de contêiner em um cluster existente.

Para ativar a verificação de configuração de workload, execute:

```bash
gcloud container clusters update NOME_DO_CLUSTER \
  --region=REGIAO_DO_COMPUTE \
  --security-posture=standard
```

Substitua:

- `NOME_DO_CLUSTER`: o nome do cluster.
- `REGIAO_DO_COMPUTE`: a região do Compute Engine do cluster.

Para clusters Standard zonais, utilize `--zone=ZONA_DO_COMPUTE`.

### Validando a ativação

Para validar via CLI se tudo foi ativado corretamente, execute:

```bash
gcloud container clusters describe NOME_DO_CLUSTER \
  --region=REGIAO_DO_COMPUTE | grep securityPostureConfig -A 2
```

O output deve ser parecido com:

```yaml
securityPostureConfig:
  mode: BASIC
  vulnerabilityMode: VULNERABILITY_MODE_UNSPECIFIED
```

Para validar pelo console do Google Cloud, vá em **Kubernetes Engine > Clusters**, clique no nome do cluster, abra a seção **Segurança** e verifique se o item **Postura de segurança** aparece como ativado.

## Criando um workload para teste

Para testar, vamos criar um Deployment com a imagem `busybox`, violando intencionalmente alguns padrões de segurança de Pod.

Salve o manifesto abaixo como `app-problema.yaml` no Cloud Shell:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-problema
  labels:
    app: problema
spec:
  selector:
    matchLabels:
      app: problema
      tier: web
  template:
    metadata:
      labels:
        app: problema
        tier: web
    spec:
      containers:
        - name: appbusy-problema
          image: busybox:1.28
          command: ["sh", "-c", "sleep 1h"]
          securityContext:
            runAsNonRoot: false
          resources:
            requests:
              cpu: 200m
```

Crie o Deployment no cluster:

```bash
kubectl apply -f app-problema.yaml
```

Output esperado:

```text
deployment.apps/app-problema created
```

Aguarde a criação do Deployment e valide:

```bash
kubectl get deploy app-problema
```

Se o output for parecido com o abaixo, significa que o Deployment está em execução:

```text
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
app-problema   1/1     1            1           87s
```

Você também pode visualizar o Deployment no console do Google Cloud em **Kubernetes Engine > Cargas de trabalho**.

## Verificando os resultados no painel de Postura de Segurança

Após a ativação, o scan pode levar cerca de 15 minutos para retornar os resultados.

Para verificar:

1. Vá em **Kubernetes Engine > Postura de segurança**.
2. Observe o painel com a visão geral de problemas, tipos, clusters e cargas de trabalho.
3. Na parte inferior do painel, revise as más configurações e vulnerabilidades descobertas.
4. Clique na aba **Problemas** para listar todos os achados.

Também é possível filtrar por gravidade, tipo de preocupação, localização do workload e cluster do GKE.

## Verificando os detalhes do problema e recomendações

Para visualizar informações detalhadas sobre uma preocupação de configuração específica, clique na linha correspondente ao achado.

Por exemplo, ao abrir a preocupação "O contêiner do pod permite o escalonamento de privilégios na execução", o painel exibe:

- Uma descrição breve do problema.
- Uma ação recomendada.
- Os recursos que precisam ser corrigidos.
- Comandos de exemplo para aplicar a correção.
- Instruções pelo console do Google Cloud, quando aplicável.

## Verificando os problemas descobertos no Cloud Logging

Por padrão, o GKE adiciona registros ao Cloud Logging para cada problema descoberto.

Para verificar esses logs:

1. Acesse o **Logs Explorer**.
2. No campo de consulta, insira:

```text
resource.type="k8s_cluster"
jsonPayload.@type="type.googleapis.com/cloud.kubernetes.security.containersecurity_logging.Finding"
jsonPayload.type="FINDING_TYPE_MISCONFIG"
```

3. Clique em **Run query**.

> Caso nenhum resultado seja retornado, confira o período da consulta.

Abaixo está um exemplo de log para a preocupação "O contêiner do pod permite o escalonamento de privilégios na execução":

```json
{
  "insertId": "1e8dghoas",
  "jsonPayload": {
    "finding": "PRIVILEGE_ESCALATION",
    "resourceName": "apps/v1/namespaces/default/Deployment/appweb-hello",
    "type": "FINDING_TYPE_MISCONFIG",
    "@type": "type.googleapis.com/cloud.kubernetes.security.containersecurity_logging.Finding",
    "state": "ACTIVE",
    "severity": "SEVERITY_MEDIUM"
  },
  "resource": {
    "type": "k8s_cluster",
    "labels": {
      "project_id": "615537239033",
      "location": "us-south1",
      "cluster_name": "gkesp-cluster"
    }
  },
  "timestamp": "2023-08-22T19:23:48.526604578Z",
  "severity": "INFO",
  "logName": "projects/gkesecurityposture-demo/logs/containersecurity.googleapis.com%2Ffinding",
  "receiveTimestamp": "2023-08-22T19:23:49.138710074Z"
}
```

## Resolvendo problemas de configuração

Agora vamos corrigir os problemas encontrados anteriormente.

Edite o manifesto `app-problema.yaml` e configure o `securityContext` da seguinte forma:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-problema
  labels:
    app: problema
spec:
  selector:
    matchLabels:
      app: problema
      tier: web
  template:
    metadata:
      labels:
        app: problema
        tier: web
    spec:
      containers:
        - name: appbusy-problema
          image: busybox:1.28
          command: ["sh", "-c", "sleep 1h"]
          securityContext:
            runAsUser: 1000
            runAsNonRoot: true
            allowPrivilegeEscalation: false
          resources:
            requests:
              cpu: 200m
```

O que mudou:

- `spec.containers[*].securityContext.runAsUser: 1000`: define que os processos do contêiner devem executar com o user ID 1000.
- `spec.containers[*].securityContext.runAsNonRoot: true`: resolve a preocupação de o contêiner poder executar como root.
- `spec.containers[*].securityContext.allowPrivilegeEscalation: false`: resolve a preocupação de escalonamento de privilégios na execução.

Aplique as alterações:

```bash
kubectl apply -f app-problema.yaml
```

O output deve ser parecido com:

```text
deployment.apps/app-problema configured
```

Aguarde cerca de 15 minutos para que a verificação seja concluída e o painel seja atualizado.

## Verificando logs de remediação no Cloud Logging

Também podemos usar o Cloud Logging para verificar quando os problemas foram remediados.

No Logs Explorer, use a consulta:

```text
resource.type="k8s_cluster"
jsonPayload.@type="type.googleapis.com/cloud.kubernetes.security.containersecurity_logging.Finding"
jsonPayload.type="FINDING_TYPE_MISCONFIG"
jsonPayload.state="REMEDIATED"
```

Então você deve ver registros de remediação. Abaixo está um exemplo de log remediado para a preocupação de escalonamento de privilégios:

```json
{
  "insertId": "17kld9san",
  "jsonPayload": {
    "eventTime": "2023-08-22T19:53:50.692473470Z",
    "state": "REMEDIATED",
    "resourceName": "apps/v1/namespaces/default/Deployment/appweb-hello",
    "severity": "SEVERITY_MEDIUM",
    "type": "FINDING_TYPE_MISCONFIG",
    "finding": "PRIVILEGE_ESCALATION",
    "@type": "type.googleapis.com/cloud.kubernetes.security.containersecurity_logging.Finding"
  },
  "resource": {
    "type": "k8s_cluster",
    "labels": {
      "cluster_name": "gkesp-cluster",
      "location": "us-south1",
      "project_id": "615537239033"
    }
  },
  "timestamp": "2023-08-22T19:53:50.711924469Z",
  "severity": "INFO",
  "logName": "projects/gkesecurityposture-demo/logs/containersecurity.googleapis.com%2Ffinding",
  "receiveTimestamp": "2023-08-22T19:53:50.928852982Z"
}
```

## Conclusão

Para quem deseja melhorar a segurança das cargas de trabalho sem adicionar muita complexidade, a Postura de Segurança do GKE ajuda a fazer o básico bem feito de forma nativa da nuvem e com boa integração ao ecossistema do Google Cloud.

O painel é intuitivo, direto ao ponto e os problemas encontrados têm um bom nível de detalhamento, o que facilita o entendimento e a correção.

Na época em que escrevi este texto, ainda havia pontos de melhoria:

- O tempo de verificação levava cerca de 15 minutos.
- Uma visão sobre problemas corrigidos diretamente no painel ajudaria a acompanhar a evolução da postura de segurança em ambientes produtivos com muitos clusters.
- Para saber a hora exata em que um problema foi remediado, era necessário consultar os logs de remediação no Cloud Logging.

Mesmo com esses pontos, gostei muito dessa adição ao GKE. O Google Cloud segue evoluindo seus produtos, especialmente os focados em segurança, e esse recurso é um bom exemplo de como aproximar recomendações práticas do fluxo operacional de quem administra clusters Kubernetes.

## Referências

- [GKE Security Posture dashboard now generally available with enhanced features](https://cloud.google.com/blog/products/identity-security/gke-security-posture-now-generally-available-with-enhanced-features){:target="_blank" rel="noopener noreferrer"}
- [About the security posture dashboard](https://cloud.google.com/kubernetes-engine/docs/concepts/about-security-posture-dashboard){:target="_blank" rel="noopener noreferrer"}
- [About Kubernetes security posture scanning](https://cloud.google.com/kubernetes-engine/docs/concepts/about-configuration-scanning){:target="_blank" rel="noopener noreferrer"}
- [Automatically scan workloads for configuration issues](https://cloud.google.com/kubernetes-engine/docs/how-to/protect-workload-configuration){:target="_blank" rel="noopener noreferrer"}
- [gcloud container clusters create](https://cloud.google.com/sdk/gcloud/reference/container/clusters/create){:target="_blank" rel="noopener noreferrer"}
- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/){:target="_blank" rel="noopener noreferrer"}
