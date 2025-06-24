# Estudo de Políticas de Escalonamento `Scale Out` e `Scale In` na AWS
## ✏️ Atividades:
* Criação de uma VPC com duas subnets públicas e privadas;
* Criação de um Security Group genérico;
* Criação de um Classic Load Balancer;
* Criação de um Auto Scaling Group;
* Criação de Políticas de escalonamento para criação e remoção de instâncias com base em número de requisições.

## 🔒 Criação da VPC:

### Navegue no console da AWS até `VPC`, clique em `Create VPC` e siga as seguintes configurações:

* Escolha um nome para sua VPC;
* `IPv4 CIDR block`, `IPv6 CIDR block` e `Tenancy` pode manter os valores default;
* `Number of Availability Zones (AZs)`: 2;
* `Number of public subnets`: 2;
* `Number of private subnets` 2;
* `NAT gateways ($)`: None;
* `VPC endpoints`: None;
* `DNS options`: 
    - Enable DNS hostnames ✅
    - Enable DNS resolution ✅
* Clique em `Create VPC`.

## 🔐 Criação do Security Group:

### Navegue no console da AWS até `EC2` e no painel localizado na esquerda selecione `Security Groups`, clique em `Create Security Group`:

* Como o propósito é testar as políticas de escalonamento, podemos criar um `Security Group` genérico com as seguintes `Inbound Rules`:

    | Type  | Protocol | Port Range | Source Type |  Source   |
    | ----- | -------- | ---------- | ----------- | --------- |
    | HTTP  |   TCP    |    80      |  Anywhere   | 0.0.0.0/0 |

## 🚥 Criação do Load Balancer:

### Retorne à página da `EC2` e, no menu lateral esquerdo, selecione `Load Balancers` e então clique em `Create load balancer`:

* Serão apresentadas 3 opções principais de `Load Balancer` e uma opção legado, o `Classic Load Balancer`, clique nele para expandir:
![createLoadBalancer](./images/createLoadBalancer.png)
* Em seguida clique em `Create`:
![createCLB](./images/createCLB.png)
* Em `Basic configuration` replique o que seguinte:
  * Defina um nome;
  * Em `Scheme` selecione `Internet-facing`;
* Em `Network mapping` selecione sua VPC, marque a checkbox de ambas `AZs` e selecione a subnet pública de cada:
![clb-network-config](./images/clb-network-config.png)
* Em `Security Groups` remova o default e adicione o genérico que foi criado;
* As configurações restantes podem ser mantidas as `default`.

> [!NOTE]
> A instâncias alvo serão definidas posteriormente durante a criação do `Auto Scaling Group`.

## 📈 Criação do Auto Scaling Group (ASG):

### Retorne à página da `EC2` e, no menu lateral esquerdo, selecione `Auto Scaling Groups` e clique em `Create Auto Scaling Group`:

* Nesta primeira etapa, dê um nome para seu `Auto Scaling Group`;
* Em `Launch Template` selecione a opção para criar um novo (será aberto em uma aba nova);

#### Criação do Launch Template:

* Aqui basicamente será feita a configuração de nossas EC2s;
* Dê um nome e uma versão para seu `Launch Template`;
* Em `Launch template contents`, para o Sistema Operacional, clique em `Quick start` e selecione `Amazon Linux` (por default virá a versão 2023 que é gratuita);
* Em `Instance type` selecione a `t2.micro`;
* Para `Key pair` pode manter a opção `Don't include in launch template`;
* Em `Network settings` não há necessidade de seleções no momento, a escolha de `subnet` e `Security group` pode ser feita posteriormente durante a criação do `Auto Scaling Group`, mas caso queira, pode-se selecionar logo o `Security Group`;
* Em `Storage` pode-se manter o default;
* Clique para expandir a aba de `Advanced details`, desça até o final da página, localize o campo para inserção do `User data` e insira os comandos desejados;

> [!NOTE]
> User data utilizado [aqui](https://github.com/PedroMak/ScalingPoliciesStudy/blob/master/userdata.sh).

* Clique em `Create launch template`.

#### De volta à criação do Auto Scaling Group:

* Com o nome já definido, selecione o `Launch Template` que foi criado e clique em `Next`;
* Na segunda etapa, em `Network`, selecione sua VPC, as duas subnets públicas e pode manter `Balanced best effort`. Clique em `Next`;
* Para a terceira etapa, siga as seguintes configurações:
  * Selecione `Attach to an existing load balancer` em `Load balancing`;
  * Selecione `Choose from Classic Load Balancers` e escolha o `Classic Load Balancer` que foi criado;
  * Para `VPC Lattice integration options` pode manter `No VPC Lattice service`;
   * Em `Health Checks` marque o checkbox de `Turn on Elastic Load Balancing health checks` conforme a imagem:
   ![healthcheck-config](./images/ELB-healthcheck-config.png)
* Na quarta etapa definimos as seguintes condigurações:
  * `Desired capacity`: 2;
  * `Min desired capacity`: 1;
  * `Max desired capacity`: 4;
  * Em `Automatic scaling` podemos manter `No scaling policies`;
  * As configurações restantes podem ser mantidas como default, então clique em `Next`;
* A quinta e sexta etapas são opcionais, clique em `Next` nas duas e então clique em `Create Auto Scaling Group`;

## 🚨 Criação dos alarmes:

### Navegue no console da AWS até `CloudWatch`:

#### Primeiro criaremos a política de `Scale Out` para lançamento de novas instâncias:

* No menu lateral esquerdo selecione `All metrics`, clique em `ELB`, depois em `Per-LB Metrics`;
* Em seguida, localize a métrica `RequestCount` de seu `Classic Load Balancer`:
![reqcount-metric](./images/reqcount-metric.png)
* Com ele selecionado, clique em `Create Alarm`;

> [!IMPORTANT]
> Certifique-se que o campo `Namespace` contém `AWS/ELB`.

* Em `Metric`, selecione `Sum` para `Statistics` e `1 minute` para `Period`;
* Em `Conditions` replique a seguinte configuração:
![scaleOut-conditions](./images/scaleOut-conditions.png)
* Clique em `Next`;
* Na segunda etapa serão definidas as ações:
  * Em `Notification` mantenha as confiugurações `default` e selecione a opção padrão em `Send a notification to…`;
  * Em `Auto Scaling action` replique o seguinte:
    * Em `Alarm state trigger` selecione `In Alarm`;
    * Em `Resource type` selecione `EC2 Auto Scaling group`;
    * Em `Select a group` selecione seu `Auto Scaling Group`;
    * Em `Take the following action...` selecione `ScaleOut (Add 2 instances)`;
    * Clique em `Next`;
* Na terceira etapa dê um nome e uma descrição para seu alarme e clique em `Next`;
* Revise seu alarme e, para finalizar, clique em `Create alarm`

#### Para a criação da política de `Scale In`, repita o processo, mas com as seguinte alterações:

*  Na primeira etapa, em `Conditions`, replique a seguinte configuração:
![scaleIn-conditions](./images/scaleIn-conditions.png)

* Na segunta etapa, em `Auto Scaling action` replique o seguinte:
  * Em `Alarm state trigger` selecione `In Alarm`;
  * Em `Resource type` selecione `EC2 Auto Scaling group`;
  * Em `Select a group` selecione seu `Auto Scaling Group`;
  * Em `Take the following action...` selecione `ScaleIn (Remove 2 instances)`;
* Etapas e configurações restantes podem ser feitas da mesma maneira.