# Terraform Workshop
Es un aprovisionador de infraestructura, puede almacenar infraestructura cloud configurada como código​

  - Es similar a CloudFormation en AWS, Deploy Manager de GCP o Azure Automation, pero agnóstico a algún proveedor.

# ¡A descargarlo!

  - Ingresa acá para descargar el binario https://www.terraform.io/downloads.html
  - Instalalo acorde lo requiera tú SO
  - Instala **az-cli** https://docs.microsoft.com/en-us/cli/azure/install-azure-cli

# Configuremos nuestra IaC

Empecemos declarando el proveedor de nuestro servicio, en este caso **Azure**


`provider.tf`
```
provider "azurerm" {
  features {}
}
```
Una vez declarado este archivo, podremos inicializar nuestro proyecto. Esto conectará el proyecto con el API del proveedor mediante los plugins que requiera.

```
$ terraform init
```
En caso de tener problemas inicializando el repositorio, declarar el siguiente archivo. Este fijará una versión para el proveedor.

Ya inicializado, declaremos nuestros recursos como código en el control de versiones correspondiente. 

`vm.tf`
```
variable "prefix" {
  default = "tfvmex"
}

resource "azurerm_resource_group" "main" {
  name     = "${var.prefix}-resources"
  location = "West US 2"
}

resource "azurerm_virtual_network" "main" {
  name                = "${var.prefix}-network"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_subnet" "internal" {
  name                 = "internal"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefix     = "10.0.2.0/24"
}

resource "azurerm_network_interface" "main" {
  name                = "${var.prefix}-nic"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  ip_configuration {
    name                          = "testconfiguration1"
    subnet_id                     = azurerm_subnet.internal.id
    private_ip_address_allocation = "Dynamic"
  }
}

resource "azurerm_virtual_machine" "main" {
  name                  = "${var.prefix}-vm"
  location              = azurerm_resource_group.main.location
  resource_group_name   = azurerm_resource_group.main.name
  network_interface_ids = [azurerm_network_interface.main.id]
  vm_size               = "Standard_DS1_v2"

  # Uncomment this line to delete the OS disk automatically when deleting the VM
  # delete_os_disk_on_termination = true

  # Uncomment this line to delete the data disks automatically when deleting the VM
  # delete_data_disks_on_termination = true

  storage_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "16.04-LTS"
    version   = "latest"
  }
  storage_os_disk {
    name              = "myosdisk1"
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = "Standard_LRS"
  }
  os_profile {
    computer_name  = "hostname"
    admin_username = "testadmin"
    admin_password = "Password1234!"
  }
  os_profile_linux_config {
    disable_password_authentication = false
  }
  tags = {
    environment = "staging"
  }
}
```

`mysql.tf`
```
resource "azurerm_mysql_server" "example" {
  name                = "example-mysqlserver"
  location            = "West US 2"
  resource_group_name = "tfvmex-resources"

  administrator_login          = "mysqladminun"
  administrator_login_password = "H@Sh1CoR3!"

  sku_name   = "B_Gen5_2"
  storage_mb = 5120
  version    = "5.7"

  auto_grow_enabled                 = true
  backup_retention_days             = 7
  geo_redundant_backup_enabled      = false
  infrastructure_encryption_enabled = false
  #public_network_access_enabled     = false
  ssl_enforcement_enabled           = true
  ssl_minimal_tls_version_enforced  = "TLS1_2"
}

resource "azurerm_mysql_database" "example" {
  name                = "exampledb"
  resource_group_name = "tfvmex-resources"
  server_name         = azurerm_mysql_server.example.name
  charset             = "utf8"
  collation           = "utf8_unicode_ci"
}
```

Al haber declarado todos nuestros recursos, ejecutamos
```
$ terraform plan
```
Esto para ver el plan sugerido de creación de recursos por parte de Terraform; esto puede ser visto como un preview de la infraestructura a crearse, pero no ejecutará nada más que el preview.

Si estamos conformes con lo que nos muestra, procedemos a ejecutar

```
$ terraform apply
```

Este por defecto te muestra un output identico al del plan, y luego te solicita un input, `yes` en este caso, para proceder en caso de estar de acuerdo con lo que se ve en el output.

Al ejecutar el comando, el aprovisionamiento de los recursos empezará y te irá mostrando el avance en la creación de los disintos recursos.

Al culminar el aprovisionamiento te notificará, como también en el caso de que ocurra un error inesperado.

Despleguemos otro recurso en el mismo grupo de recursos que ya hemos creado.
`aks.tf`
```
resource "azurerm_kubernetes_cluster" "example" {
  name                = "example-aks1"
  location            = "West US 2"
  resource_group_name = "tfvmex-resources"
  dns_prefix          = "exampleaks1"

  default_node_pool {
    name       = "default"
    node_count = 1
    vm_size    = "Standard_D2_v2"
  }

  identity {
    type = "SystemAssigned"
  }

  tags = {
    Environment = "Production"
  }
}

output "client_certificate" {
  value = azurerm_kubernetes_cluster.example.kube_config.0.client_certificate
}

output "kube_config" {
  value = azurerm_kubernetes_cluster.example.kube_config_raw
}
```
Con estos pocos archivos, hemos creado:

+ VNET
+ Grupo de recursos
+ VM
+ DB
+ Cluster de K8s

Esto puede ser utilizado para 
Listo por acá
![Alt Text](https://media.giphy.com/media/12NUbkX6p4xOO4/giphy.gif)

En caso de que quieras destruir recursos dentro de **terraform**
```
$ terraform destroy
o
$ terraform destroy --target NOMBRE_DEL_RECURSO 
```
