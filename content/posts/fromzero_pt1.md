+++
categories = ["devops", "noops", "gitops", "oam"]
date = "2021-01-17"
description = "material para estudo próprio e para quem se interessar ;)"
featured = "https://cavarsaio.s3-sa-east-1.amazonaws.com/post1-min-min.jpg"
featuredalt = ""
featuredpath = "date"
linktitle = ""
title = "from zero to alguma coisa - criando uma plataforma na aws com kubernetes e outras paradinhas (parte 1/n)"
slug = "from zero to alguma coisa - criando uma plataforma na aws com kubernetes e outras paradinhas"
type = "post"
+++

## Propósitos e o motivo de eu estar começando esse humilde blog

Bom, ja faz um tempo que to querendo escrever alguma coisa sobre o que tenho estudado e trabalhado no dia-a-dia, acho que o dia chegou!! ja adianto que não sou um cara muito didático, mas vou me esforçar pra escrever o mais claro possível. Feedbacks e sugestões são muito bem vindos :D

a idéia desse tutorial é mostrar a criação de uma plataforma com kubernetes na aws, utilizando EKS, terragrunt, argoCD, crossplane, openfaas e outras ferramentinhas que tão na moda aí, não tenho muita idéia de como vai ficar a plataforma final, essa é a primeira parte e vou fazendo conforme for estudando, isso quer dizer que pode ter muitas partes (por isso o parte 1/n), porém a minha vontade no final é ter uma plataforma inteira orquestrada com controllers dentro do kubernetes e todo fluxo passar por la.

e também é importante deixar claro aqui que muito desse conhecimento está sendo adquirido em tempo de build (do post), e também muitos pontos aqui uso e aprendo no dia-a-dia no iti com o pessoal da equipe em que faço parte, então ja fica aqui meu agradecimento ao pessoal da squad cloud-noops, pessoal é muito foda mesmo! 

vou tentar fazer um post desses por semana, mas não prometo pois a procrastinação ja faz parte de mim e também depende de como estiver a correria por aqui para dar vazão pra escrever os próximos passos. 

## Bora lá! (um pouquinho mais de teoria, sorry)

Estava pensando aqui como iria começar essa plataforma, e uma das minhas "premissas" vai ser ter um ambiente inteiro como código, pra eu poder destruir e recriar a hora que eu quiser, isso pq deixar um ambiente em pé na AWS custa caro e eu ainda não ganhei na loteria...

pra isso, vou usar terraform pra subir a primeira plataforma, como no iti estamos usando bastante terragrunt, vou usar ele pra escrever toda a base do kubernetes, addons e configurações, nessa primeira parte vou tentar deixar toda essa base pronta, pra nos proximos post configurar todos os addons, 1 por 1 (ou talvez 2 por post)

Pensando na especificação OAM e em deixar o caminho pronto pra implementar o GitOps, fiz a definição de algumas ferramentas pra começar, é o que vamos falar nos próximos tópicos

### Plataforma (Kubernetes)

Pra plataforma não vou perder tanto tempo deployando o cluster com kops, kubespray etc, vou pelo caminho mais simples, até pq o intuito desse post não é conhecer o kubernetes a fundo e sim construir uma plataforma completa com addons e um fluxo pra deploy das apps. Por isso vou usar o EKS e o módulo publico de terraform pra isso (mais pra frente falamos disso)

Como disse no tópico anterior, vou utilizar o terragrunt pra conseguir subir as nossas VPCs, EKS, Addons e tudo mais, e pra isso vou me basear (copiar?) muito do estilo do projeto TEKS https://github.com/particuleio/teks, esse repositório vale muita a pena dar uma olhada, ele é bem organizado e deu muitas idéias pra equipe enquanto estávamos em um projeto bem foda aqui no iti.

Pro fluxo de CI dessa plataforma vou usar o Github Actions, mas isso vai ser apenas pra aplicar o terragrunt na minha conta da AWS, os pipelines de aplicação vão ficar por conta do Tekton! e a pipeline com github actions deve ficar pro próximo post...

### Addons

Aqui entra uma parte bem legal, que é o que vamos deployar no nosso cluster, alguns ja tem pronto no repositório do TEKS, outros vamos adicionar quando fizermos nosso fork, e esses são alguns do que estava pensando em deployar junto ao cluster:

- *node-termination-handler*: pra econimizar custos, em algum momento vou fazer o deploy com nodes spots, e essa aplicação ajuda a gente a gerenciar essas instancias spot com alertas, draining dos nodes, entre outros

- *cluster-autoscaler*: esse cara vai controlar o autoscale do cluster, para além de ter nodes spots, não desperdiçarmos recursos enquanto usamos a plataforma.

- *prometheus-operator*: operator pra controlar o deploy do prometheus no cluster e entregar alguns recursos importantes para monitorarmos as apps.

- *tekton*: vamos criar nossas pipelines com o tekton

- *argoCD*: o argoCD vai ser responsável por ajudar a gente a implementar o gitops, fazendo o reconcilitation de todos os recursos que vamos deployar no nosso cluster.

- *nginx ingress*: vamos usar o nginx ingress para todo trafégo externo pro cluster

- *external dns*: em conjunto com o ingres, o external dns vai fazer todo o cadastro automáticos de recordsets na zona DNS do route53

- *crossplane*: o fluxo de deploy da base da plataforma vai ser com terragrunt, porém a idéia é fazer o deploy dos recursos das apps com o crossplane, isso vai ajudar a gente ter os resources da aws que as apps usam declarados como objetos no kubernetes e implementar um pouco do OAM.

- *openfaas*: hj uma dificuldade que temos no dia-a-dia é manter um fluxo separado pra lambdas e depender do cloud provider pra isso, a idéia com o openfaas é ter as nossas funções também declaradas no nosso cluster como objetos e com isso abstrair o cloudprovider pra esses deploys tbm.


Observação bem importante!: posso adicionar/remover esses addOns conforme for seguindo nos posts, pois muita coisa pode mudar e algumas coisas podem fazer sentido adicionar ou remover algumas.


## Mão na Massa

Vou começar com os repositórios que vão ser utilizados pra essa primeira versão: 

- Plataforma: https://github.com/hcavarsan/base-platform

Aqui a idéia é ter todo o código da plataforma, e como disse, muita coisa vou me inspirar no TEKS, a estrutura de diretórios inicialmente vai ficar assim: 

```
├── README.md
└── platform
    ├── terragrunt.hcl
    └── sa-east-1
        ├── addons
        │   └── terragrunt.hcl
        ├── eks
        │   └── terragrunt.hcl
        └── vpc
            └── terragrunt.hcl
```


- Addons: https://github.com/hcavarsan/terraform-kubernetes-addons

Esse projeto basicamente é um fork do modulo: https://github.com/particuleio/terraform-kubernetes-addons, vamos aproveitar alguns addons porém vamos adicionar mais alguns outros para conseguir seguir com a proposta de addons apresentada no tópico anterior 


nos próximos tópicos vou explicar como ficou cada repositório e ja fazendo o deploy de toda a base que vamos utilizar pra começar a fuçar.

## Configuração de Addons:

Repo/Fork: https://github.com/particuleio/terraform-kubernetes-addons

Aqui temos quase todos os addons que precisamos para deployar nossa plataforma, nele é utilizado o helm_release do terraform para aplicar os modulos, e basicamente precisamos aplicar esse modulo e nos inputs apontar quais recursos queremos deployar, e se precisarmos passar alguns values, exemplos:

simples:
```
  external_dns = {
    enabled = true
  }
```

passando values a mais para aplicar os charts:
```
  external_dns = {
    enabled = true
    extra_values = <<-EXTRA_VALUES
      image:
        repository: xxxxxxx
      EXTRA_VALUES
  }
```

tbm tem alguns addons que vamos precisar adicionar nesse módulo pois ainda não tem, são eles:

- crossplane
- tekton
- openfaas
- hpa operator
- argoCD

a idéia é deixar essa configuração pra próxima parte dessa jornada, mas é bom ja termos isso mapeado!
## Deploy da Stack

Repo: https://github.com/hcavarsan/base-platform

Bom, ja deixei pronto toda a estrutura do Terragrunt, que basicamente se resume em alguns arquivos: 

- *platform/terragrunt.hcl*: basicamente aqui tem toda a configuração de bucket e lock de dynamo que precisamos, como são informações "comuns" a todos os modulos, podemos deixar elas nesse arquivo e "importar" nos arquivos .hcl que aplica cada modulo que precisamos.

- *platform/sa-east-1/vpc/terragrunt.hcl*: aqui efetivamente estamos aplicando  o modulo oficial de vpc, e configurando todas as subnets, publica, interna e privada

- *platform/sa-east-1/eks/terragrunt.hcl*: aqui aplicamos o EKS nas subnets da VPC criada anteriormente, habilitamos IRSA (vamos falar disso mais pra frente) e configuramos os node groups(por enquanto sem spots)

- *platform/sa-east-1/addons/terragrunt.hcl*: aqui subimos os addons que precisamos utilizando o nosso módulo, definindo o que queremos habilitar e passando alguns values especificos para os charts que precisam ser customizados

com isso explicado, vamos subir nosso cluster! exportando as variaveis de credenciais da AWS e subindo esse terragrunt, devemos ter nosso ambiente pronto!, o terragrunt se encarrega de criar o bucket, dynamos e subir todos recursos na ordem que precisar, então é só rodar os comandos abaixo que nosso cluster deve subir (pode ir tomar um café enquanto roda pq demora bem):

```
export AWS_ACCESS_KEY_ID="XXXXXXXXXXX"
export AWS_SECRET_ACCESS_KEY="XXXXXXXXXX"
terragrunt plan-all
terragrunt apply-all
````

Ah!! e para acessar o cluster e ver que ta tudo rodando, é só dar um `aws eks --region sa-east-1 update-kubeconfig --name cluster-default` e acessar via kubectl pra se certificar que os addons estão todos lá....

tbm é legal dar uma olhada no s3 e dynamo pra ver toda a estrutura de state e lock que o terragrunt ja entrega pra gente :top:

## Concluindo a primeira parte!

Bom, acho que concluí a primeira parte que era subir um cluster 100% com código, foi pouca coisa, mas isso é o que vou usar de base para o restante de projeto, na próxima vou dar uma organizada no terragrunt pra reaproveitar algumas variáveis, fazer a pipeline para a plataforma e depois a configuração dos demais addons no módulo, 

Com isso ja vai ser possível começar a acessar as apps através do nginx-ingress e ver as paradas tomando forma o/, e depois disso é só ladeira abaixo configurando addon por addon...

Qualquer critíca é bem vinda, então sintam-se a vontade de opinar, descer a lenha e/ou sugerir melhorias pra esse post ;D 



