# Guia de Implantação da Solução no Azure Kubernetes Service (AKS)

Este guia fornece instruções detalhadas para implantar a solução de aplicação Python Flask com banco de dados Redis no Azure Kubernetes Service (AKS).

## Pré-requisitos

- Conta ativa na plataforma Microsoft Azure
- CLI do Azure instalada e configurada
- Terraform e Ansible instalados na máquina local

## Etapas de Implantação

### 1. Criar o cluster AKS usando Terraform

1. Crie um novo diretório para o projeto e navegue até ele.
2. Crie um arquivo `main.tf` e adicione o seguinte código Terraform:

   ```hcl
   provider "azurerm" {
     version = "~> 2.46.0"
     features {}
   }

   resource "azurerm_resource_group" "rg" {
     name     = "my-resource-group"
     location = "West Europe"
   }

   resource "azurerm_kubernetes_cluster" "aks" {
     name                = "my-aks-cluster"
     location            = azurerm_resource_group.rg.location
     resource_group_name = azurerm_resource_group.rg.name
     kubernetes_version  = "1.21.0"
     node_resource_group = "my-node-resource-group"

     default_node_pool {
       name       = "default"
       node_count = 3
       vm_size    = "Standard_D2_v2"
     }

     identity {
       type = "SystemAssigned"
     }
   }

   output "kubernetes_cluster_name" {
     value = azurerm_kubernetes_cluster.aks.name
   }

   output "kubernetes_cluster_region" {
     value = azurerm_kubernetes_cluster.aks.location
   }
   ```

3. Inicialize o Terraform e aplique as alterações:

   ```bash
   terraform init
   terraform apply
   ```

   Esse processo levará alguns minutos para provisionar o cluster AKS.

### 2. Criar o Azure Container Registry usando Terraform

1. Adicione o seguinte código ao final do arquivo `main.tf`:

   ```hcl
   resource "azurerm_container_registry" "acr" {
     name                = "myacrregistry"
     resource_group_name = azurerm_resource_group.rg.name
     location            = azurerm_resource_group.rg.location
     sku                 = "Standard"
   }

   output "container_registry_name" {
     value = azurerm_container_registry.acr.name
   }
   ```

2. Execute novamente `terraform apply` para criar o registro de contêiner ACR.

### 3. Implantar a aplicação Flask e o Redis no AKS usando Ansible

1. Crie um novo diretório para os arquivos Ansible e navegue até ele.
2. Crie um arquivo `playbook.yml` e adicione o seguinte código:

   ```yaml
   - hosts: localhost
     connection: local
     tasks:
       - name: Create Kubernetes namespace
         kubernetes.core.k8s:
           name: my-app
           api_version: v1
           kind: Namespace
           state: present

       - name: Deploy Flask application
         kubernetes.core.k8s:
           definition:
             apiVersion: apps/v1
             kind: Deployment
             metadata:
               name: flask-app
               namespace: my-app
             spec:
               replicas: 2
               selector:
                 matchLabels:
                   app: flask-app
               template:
                 metadata:
                   labels:
                     app: flask-app
                 spec:
                   containers:
                   - name: flask-app
                     image: myacrregistry.azurecr.io/flask-app:v1
                     ports:
                     - containerPort: 5000

       - name: Deploy Redis
         kubernetes.core.k8s:
           definition:
             apiVersion: apps/v1
             kind: Deployment
             metadata:
               name: redis
               namespace: my-app
             spec:
               replicas: 1
               selector:
                 matchLabels:
                   app: redis
               template:
                 metadata:
                   labels:
                     app: redis
                 spec:
                   containers:
                   - name: redis
                     image: redis:6.2.6
                     ports:
                     - containerPort: 6379
   ```

3. Execute o playbook Ansible:

   ```bash
   ansible-playbook playbook.yml
   ```

   Esse playbook cria um namespace `my-app`, implanta um Deployment para a aplicação Flask com 2 réplicas e um Deployment para o Redis no cluster AKS.

### 4. Expor a aplicação Flask

1. Crie um arquivo `service.yml` com o seguinte conteúdo:

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: flask-service
     namespace: my-app
   spec:
     type: LoadBalancer
     ports:
     - port: 80
       targetPort: 5000
     selector:
       app: flask-app
   ```

2. Aplique o serviço no cluster Kubernetes:

   ```bash
   kubectl apply -f service.yml
   ```

   Isso expõe a aplicação Flask através de um serviço do tipo `LoadBalancer` na porta 80.

### 5. Verificar a implantação

1. Obtenha o endereço IP público do serviço Flask:

   ```bash
   kubectl get service -n my-app
   ```

   O endereço IP público será exibido no campo `EXTERNAL-IP` para o serviço `flask-service`.

2. Acesse a aplicação Flask no navegador usando o endereço IP público.

Parabéns! Você implantou com sucesso a solução de aplicação Python Flask com banco de dados Redis no Azure Kubernetes Service (AKS).

