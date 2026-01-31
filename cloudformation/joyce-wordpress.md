# Documentação da Infraestrutura Joyce WordPress

Este documento descreve a infraestrutura provisionada pelo template CloudFormation `joyce-wordpress.yaml`. O objetivo é criar um ambiente de desenvolvimento persistente, escalável e de baixo custo utilizando instâncias Spot ARM64 na AWS.

## Visão Geral da Arquitetura

O template cria uma instância EC2 gerenciada por um Auto Scaling Group (ASG) que se configura automaticamente ao iniciar. A infraestrutura é desenhada para ser "stateless" no que diz respeito à instância (que pode ser terminada a qualquer momento), mas "stateful" no que diz respeito aos dados e configurações, que são persistidos em volumes EBS externos e buckets S3.

### Principais Componentes

1.  **Computação (EC2 Spot & Graviton)**
    *   Utiliza instâncias da família **Graviton (ARM64)** para melhor performance/custo.
    *   **Auto Scaling Group (ASG):** Mantém a quantidade desejada de instâncias (padrão 0 a 3).
    *   **Estratégia de Spot:** Configurado com `CapacityOptimized` para minimizar interrupções.
    *   **Alta Disponibilidade:** Suporta múltiplos tipos de instância (`m7g.large`, `c7g.large`, `r7g.large`) para evitar erros de `UnfulfillableCapacity`. Se um tipo não estiver disponível, o ASG tenta o próximo automaticamente.

2.  **Armazenamento Persistente**
    *   **EBS Volume Independente:** Um volume `gp3` de 8GB é criado separadamente da instância. Ele possui `DeletionPolicy: Retain`, garantindo que os dados sobrevivam mesmo se a stack for deletada. Este volume é detectado e montado automaticamente pela instância no boot em `/app`.
    *   **S3 Bucket:** Armazena configurações, scripts, chaves SSH e backups. Pode criar um novo bucket ou usar um existente. É montado no sistema de arquivos via `s3fs`.

3.  **Rede e DNS**
    *   **DNS Dinâmico (Route53):** A instância atualiza automaticamente seu próprio registro DNS (`joyce.lcdev.click`) ao iniciar, apontando para seu novo IP público.

4.  **Segurança e Permissões**
    *   **IAM Role:** A instância possui permissões administrativas (`ec2:*`, `s3:*`, `route53:*`, `cloudformation:*`) para auto-configuração e gerenciamento de recursos.

---

## Detalhes do Script de Inicialização (UserData)

O script `UserData` é executado na primeira inicialização da instância e realiza a configuração completa do ambiente de desenvolvimento ("Workspace"):

1.  **Instalação de Ferramentas Base:**
    *   Atualiza o sistema (Ubuntu 22.04).
    *   Instala `aws-cli` (v2), `s3fs`, `git`, `zip`, `unzip`.
    *   Instala e configura o **CloudWatch Agent** para métricas de memória e disco.

2.  **Montagem de Volumes:**
    *   **EBS Persistente:** Identifica o volume pelo ID, anexa à instância, formata (se necessário) e monta em `/app`.
    *   **S3 Bucket:** Monta o bucket S3 em `/tmp/${PROJECT_NAME}` para acesso rápido a arquivos de configuração.

3.  **Configuração de Desenvolvimento:**
    *   **Docker:** Instala Docker Engine e Docker Compose.
    *   **Node.js:** Instala NVM (Node Version Manager) e Node.js v20.
    *   **GitHub CLI:** Instala e configura a ferramenta `gh`.

4.  **Personalização do Usuário:**
    *   **SSH Keys:** Baixa e configura chaves SSH privadas do S3.
    *   **Dotfiles e Scripts:** Copia scripts personalizados (`gcp`, `my_commands`, etc.) e configurações de shell (`.bashrc`) do S3 para o usuário `ubuntu`.

---

## Parâmetros do Template

| Parâmetro | Descrição | Valor Padrão |
| :--- | :--- | :--- |
| `VpcId` | ID da VPC onde os recursos serão criados. | - |
| `SubnetIds` | Lista de Subnets para o Auto Scaling Group. | - |
| `ResourcePrefix` | Prefixo para nomear recursos (S3, IAM Roles, etc). | `joyce-wordpress` |
| `ExistingS3BucketName` | Nome de um bucket S3 existente (opcional). | `alchembook-workspace-bucket` |
| `KeyPair` | Nome do KeyPair EC2 para acesso SSH. | - |
| `AvailabilityZone` | AZ onde o volume persistente será criado. | `us-west-2d` |
| `ASGMinCapacity` | Capacidade mínima do ASG. | `0` |
| `ASGMaxCapacity` | Capacidade máxima do ASG. | `3` |
| `ASGDesireedCapacity` | Capacidade desejada inicial. | `0` |
| `DomainName` | Domínio DNS a ser atualizado. | `joyce.lcdev.click` |

## Recursos Criados (Resumo Técnico)

*   `AWS::EC2::Volume`: Volume de dados persistente.
*   `AWS::S3::Bucket`: Bucket de artefatos (condicional).
*   `AWS::IAM::Role` & `InstanceProfile`: Identidade da instância.
*   `AWS::EC2::LaunchTemplate`: Definição da instância (AMI, UserData, Tipo).
*   `AWS::AutoScaling::AutoScalingGroup`: Gerenciador de ciclo de vida das instâncias.

---

## Como Usar

1.  **Deploy:** Execute o template CloudFormation fornecendo a VPC e Subnets.
2.  **Acesso:** Após o deploy, ajuste a capacidade desejada do ASG para 1.
3.  **Conexão:** Aguarde alguns minutos para o script de boot rodar. Acesse via SSH ou use o DNS configurado (`joyce.lcdev.click`).
4.  **Parar:** Para economizar, defina a capacidade do ASG para 0. O volume de dados (`/app`) será preservado para o próximo uso.
