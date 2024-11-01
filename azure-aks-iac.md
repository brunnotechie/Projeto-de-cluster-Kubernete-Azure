## Infraestrutura como Código (IaC)

A infraestrutura dessa solução na nuvem da Microsoft Azure é provisionada e gerenciada usando as ferramentas de IaC Terraform e Ansible.

### Terraform

O código Terraform é responsável por criar os recursos de nível superior na plataforma Azure, como o cluster do AKS e o Azure Container Registry (ACR).

```hcl
# Provisionamento do AKS
resource "azurerm_kubernetes_cluster" "aks_cluster" {
  name                = "my-aks-cluster"
  location            = "West Europe"
  resource_group_name = "my-resource-group"
  kubernetes_version  = "1.21.0"
  node_resource_group = "my-node-resource-group"

  default_node_pool {
    name       = "default"
    node_count = 3
    vm_size    = "Standard_D2_v2"
  }
}

# Provisionamento do ACR
resource "azurerm_container_registry" "acr" {
  name                = "myacrregistry"
  resource_group_name = "my-resource-group"
  location            = "West Europe"
  sku                 = "Standard"
}
```

Esse código Terraform cria um cluster AKS com 3 nós de tamanho `Standard_D2_v2` e um registro de contêiner ACR no grupo de recursos `my-resource-group` na região "West Europe".

### Ansible

O código Ansible é responsável por implantar a aplicação Flask e o banco de dados Redis no cluster AKS provisionado pelo Terraform.

```yaml
# Playbook para implantar a aplicação Flask e Redis no AKS
- hosts: localhost
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

Esse playbook Ansible cria um namespace `my-app`, implanta um Deployment para a aplicação Flask com 2 réplicas e um Deployment para o Redis no cluster AKS.

A imagem da aplicação Flask é obtida do registro de contêiner ACR provisionado pelo Terraform, enquanto a imagem do Redis é obtida diretamente do Docker Hub.

