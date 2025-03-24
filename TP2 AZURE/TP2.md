# Part I : Programmatic approach

# I. Premiers pas

🌞 **Créez une VM depuis le Azure CLI**

- en utilisant uniquement la commande `az` donc
- assurez-vous que dès sa création terminée, vous pouvez vous connecter en SSH en utilisant une IP publique
- vous devrez préciser :
  - quel utilisateur doit être créé à la création de la VM
  - le fichier de clé utilisé pour se connecter à cet utilisateur
  - comme ça, dès que la VM pop, on peut se co en SSH !
- je vous laisse faire vos recherches pour créer une VM avec la commande `az`

````
vm create --resource-group Efrei1 -n VMtp2 --image Ubuntu2204 --admin-username micro --ssh-key-values .ssh/id_rsa.pub --public-ip-sku Standard
````

### Puis se connecter en SSH

🌞 **Assurez-vous que vous pouvez vous connecter à la VM en SSH sur son IP publique.**

- une fois connecté, observez :
  - **la présence du service `walinuxagent`**
    - permet à Azure de monitorer et interagir avec la VM

`systemctl status walinuxagent`

-> running

  - **la présence du service `cloud-init`**
    - permet d'effectuer de la configuration automatiquement au premier lancement de la VM
    - c'est lui qui a créé votre utilisateur et déposé votre clé pour se co en SSH !
    - vous pouvez vérifier qu'il s'est bien déroulé avec la commande `cloud-init status`

`cloud-init status`

-> done

# II. Un ptit LAN

🌞 **Créez deux VMs depuis le Azure CLI**

- assurez-vous qu'elles ont une IP privée (avec `ip a`)
- elles peuvent se `ping` en utilisant cette IP privée
- deux VMs dans un LAN quoi !

### Créer un réseau privé

```
network vnet create --resource-group Efrei1 --name MyVNet --address-prefixes 10.0.0.0/16 --subnet-name MySubnet --subnet-prefixes 10.0.1.0/24
```

### Création de VM1 et VM2

vm create --resource-group Efrei1 --name VM1 --image Ubuntu2204 --admin-username micro --ssh-key-values ./.ssh/id_rsa.pub --vnet-name MyVNet --sub

même commande mais en mettant `VM2`

```
micro@VM1:~$ ping 10.0.1.5
PING 10.0.1.5 (10.0.1.5) 56(84) bytes of data.
64 bytes from 10.0.1.5: icmp_seq=1 ttl=64 time=1.98 ms
64 bytes from 10.0.1.5: icmp_seq=2 ttl=64 time=1.58 ms
```

```
micro@VM2:~$ ping 10.0.1.4
PING 10.0.1.4 (10.0.1.4) 56(84) bytes of data.
64 bytes from 10.0.1.4: icmp_seq=1 ttl=64 time=0.781 ms
64 bytes from 10.0.1.4: icmp_seq=2 ttl=64 time=1.25 ms
64 bytes from 10.0.1.4: icmp_seq=3 ttl=64 time=1.18 ms
```

Part II : cloud-init

## 2. Gooooo

🌞 **Tester `cloud-init`**

- en créant une nouvelle VM et en lui passant ce fichier `cloud-init.txt` au démarrage
- pour ça, utilisez une commande `az vm create`
- utilisez l'option `--custom-data /path/to/cloud-init.txt`

Crée une fihier `cloud-init.txt`

### Et mettre dedans : 

```
#cloud-config
users:
  - default
  - name: alexa 
    sudo: ['ALL=(ALL) NOPASSWD:ALL']  
    shell: /bin/bash
    ssh_authorized_keys:
      - <clé publique>  
```

```
vm create --resource-group Efrei1 --name VMcloud --image Ubuntu2204 --admin-username micro --ssh-key-values ./.ssh/id_rsa.pub --custom-data "C:\Users\alexa\Documents\cloud-init.txt"
```
🌞 **Vérifier que `cloud-init` a bien fonctionné**

- connectez-vous en SSH à la VM nouvellement créée, directement sur le nouvel utilisateur créé par `cloud-init`

`micro@VMcloud:~$`

## 3. Write your own

🌞 **Utilisez `cloud-init` pour préconfigurer la VM :**

- installer Docker sur la machine
- ajoutez un user qui porte votre pseudo
  - il a un password défini
  - clé SSH publique déposée
  - il a accès aux droits de `root` via `sudo`
  - membre du groupe `docker`
- l'image Docker `alpine:latest` doit être téléchargée

### Dans le fichier cloud-init.txt : 

```
#cloud-config
users:
  - default
  - name: micro
    sudo: ['ALL=(ALL) NOPASSWD:ALL']  
    groups: Efrei1
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDAzqqtqpmVrP6D2CFB+mml1mvJx1l++3fuQbXQtDmPqzmxdJG1mf6G+K1eM5cuMkwOOL7bLJamouR2yh9++JCxY9s7kZLV6Y2tgAAJOQxAM2QcYgGX9YJIlYAvxzJgV6aFxbkp/yy1nPSNoPasHzVb7UUYHrqB6IcYrhjPycPASsC9AMk2ZtwavHvQNEDI7rZH0Ul2MsPMEf8sww8t+UXuTeSNrpNRevS2VZUz6RmlMJnV2zQKFlJ1vw4kmoeEvxM9oTAF1rKbaR9risDUhAQmfb8VYvASVR1jCs9nLBJhyNiim+z1rwc7BEUCoEVb4CFtHsCs8gdIipepvyumnPtr+42kXV8n2H6bs+HoPiUwElNfrv/hiD8ZmH6mtwLlDmaiQzQgthwL3zxdIZxLQ2VQBefpottEBFAEek68tLBOoz+fJbLv71bHnjKfTP0bhZ9LzQ6TAbhZCzsTZkbkCU7eyFxlzCIdKkWsspTWFIwLaqT8CMi+e1APy4OI4tOU43UWxkoSUi4vUbHkh6izW2nWGWrTlruVG7LgsPDL2fyq6XKFbxtr1ULm5LJjVZrVo8ebLsAjwTWPGnMm5kc1XvKoqhYUVgaWlmUfBG13QolJ2EU/J8UN+TpgxjdZHzxVGwnt+XWgmhdSMIYdtlyvL+QJ12mNJhjNttj3eeUdlbQfkw== alexa@PAVASUS
    lock_passwd: false
    passwd: $6$m0GK/yFWuYflgm/9$D.yLWYSuKQq2UasnQyXu2URxh3u3tMxoJPHEbwUgyh0nbtrZOcPlE58ULeSXq70hQOB/W0xDoAzw41hgehgRo.

package_update: true
package_upgrade: true
packages:
  - docker.io  

runcmd:
  - systemctl enable docker  
  - systemctl start docker  
  - docker pull alpine:latest  
```
### Crée une nouvelle machine puis se connecter en ssh : 

```
micro@VMcloud-docker:~$ sudo docker images
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
alpine       latest    aded1e1a5b37   4 weeks ago   7.83MB
```
---


# Part III : Terraform



🌞 **Constater le déploiement**

- depuis la WebUI si tu veux
- pour le compte-rendu : depuis le CLI `az`
  - `az vm list`
  - `az vm show --name VM_NAME --resource-group RESOURCE_GROUP_NAME`
  - `az group list`
  - n'oubliez pas que vous pouvez ajouter `-o table` pour avoir un output plus lisible par un humain :)

`az>> vm list -o table`

## 3. Do it yourself

🌞 **Créer un *plan Terraform* avec les contraintes suivantes**

- `node1`
  - Ubuntu 22.04
  - 1 IP Publique
  - 1 IP Privée
- `node2`
  - Ubuntu 22.04
  - 1 IP Privée
- les IPs privées doivent permettre aux deux machines de se `ping`

⚠️⚠️⚠️ **Je vous recommande TRES fortement de changer le préfixe que vous avez choisi dans le fichier `variables.tf` (pour chaque nouveau plan Terraform).

> Pour accéder à `node2`, il faut donc d'abord se connecter à `node1`, et effectuer une connexion SSH vers `node2`. Vous pouvez ajouter l'option `-j` de SSH pour faire ~~des dingueries~~ un rebond SSH (`-j` comme Jump). `ssh -j node1 node2` vous connectera à `node2` en passant par `node1`.

---

- ### ``Main.tf``
- ````bash
    provider "azurerm" {
    features {}
  }

  resource "azurerm_resource_group" "rg" {
    name     = var.resource_group_name
    location = var.location
  }

  resource "azurerm_virtual_network" "vnet" {
    name                = var.vnet_name
    location            = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name
    address_space       = ["10.0.0.0/16"]
  }

  resource "azurerm_subnet" "subnet" {
    name                 = var.subnet_name
    resource_group_name  = azurerm_resource_group.rg.name
    virtual_network_name = azurerm_virtual_network.vnet.name
    address_prefixes     = ["10.0.1.0/24"]
  }

  resource "azurerm_public_ip" "node1_public_ip" {
    name                = "node1PublicIP"
    location            = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name
    allocation_method   = "Dynamic"
  }

  resource "azurerm_network_interface" "node1_nic" {
    name                = "node1NIC"
    location            = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name

    ip_configuration {
      name                          = "internal"
      subnet_id                     = azurerm_subnet.subnet.id
      private_ip_address_allocation = "Dynamic"
      public_ip_address_id          = azurerm_public_ip.node1_public_ip.id
    }
  }

  resource "azurerm_network_interface" "node2_nic" {
    name                = "node2NIC"
    location            = azurerm_resource_group.rg.location
    resource_group_name = azurerm_resource_group.rg.name

    ip_configuration {
      name                          = "internal"
      subnet_id                     = azurerm_subnet.subnet.id
      private_ip_address_allocation = "Dynamic"
    }
  }

  resource "azurerm_linux_virtual_machine" "node1" {
    name                = var.node1_name
    resource_group_name = azurerm_resource_group.rg.name
    location            = azurerm_resource_group.rg.location
    size                = "Standard_B1s"
    admin_username      = var.admin_username
    network_interface_ids = [azurerm_network_interface.node1_nic.id]

    admin_ssh_key {
      username   = var.admin_username
      public_key = file("~/.ssh/id_rsa.pub")
    }

    os_disk {
      caching              = "ReadWrite"
      storage_account_type = "Standard_LRS"
    }

    source_image_reference {
      publisher = "Canonical"
      offer     = "0001-com-ubuntu-server-jammy"
      sku       = "22_04-lts"
      version   = "latest"
    }
  }

  resource "azurerm_linux_virtual_machine" "node2" {
    name                = var.node2_name
    resource_group_name = azurerm_resource_group.rg.name
    location            = azurerm_resource_group.rg.location
    size                = "Standard_B1s"
    admin_username      = var.admin_username
    network_interface_ids = [azurerm_network_interface.node2_nic.id]

    admin_ssh_key {
      username   = var.admin_username
      public_key = file("~/.ssh/id_rsa.pub")
    }

    os_disk {
      caching              = "ReadWrite"
      storage_account_type = "Standard_LRS"
    }

    source_image_reference {
      publisher = "Canonical"
      offer     = "0001-com-ubuntu-server-jammy"
      sku       = "22_04-lts"
      version   = "latest"
    }
  }
  ````

- ### ``Variables.tf``

```
variable "prefix" {
  description = "Préfixe pour le nom des ressources"
  default     = "tp2-terraform-nodes"  
}

variable "location" {
  description = "Région Azure"
  default     = "West Europe"
}
```

- ### ``Outputs.tf``

- ````bash
    output "node1_public_ip" {
    value = azurerm_public_ip.node1_public_ip.ip_address
  }

  output "node1_private_ip" {
    value = azurerm_network_interface.node1_nic.private_ip_address
  }

  output "node2_private_ip" {
    value = azurerm_network_interface.node2_nic.private_ip_address
  }
  ````

````bash
PS C:\Users\antoi> ssh -J antoine@52.178.187.101 antoine@10.0.1.5
The authenticity of host '10.0.1.5 (<no hostip for proxy command>)' can't be established.
ED25519 key fingerprint is SHA256:gVUVRTkvAfVFAdKWr2OYYH1539uJJpXkqNolDnETj0M.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
````
- ### ``Ping`` :

````bash
micro@CLOUD--node2-vm:~$ ping 10.0.1.4
PING 10.0.1.4 (10.0.1.4) 56(84) bytes of data.
64 bytes from 10.0.1.4: icmp_seq=1 ttl=64 time=1.35 ms

micro@CLOUD--node1-vm:~$ ping 10.0.1.5
PING 10.0.1.5 (10.0.1.5) 56(84) bytes of data.
64 bytes from 10.0.1.5: icmp_seq=1 ttl=64 time=1.15 ms
````

## 4. cloud-iniiiiiiiiiiiiit

### A. Un premier tf + cloud-init

🌞 **Intégrer la gestion de `cloud-init`**

`variables.tf`
```
variable "prefix" {
  description = "Project prefix"
  default     = "partie3"
}

variable "location" {
  description = "Azure region"
  default     = "West Europe"
}
```
`main.tf`
```
provider "azurerm" {
  features {}
  subscription_id = "..."
}

resource "azurerm_resource_group" "main" {
  name     = "${var.prefix}-resources"
  location = var.location
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
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_public_ip" "pip" {
  name                = "${var.prefix}-pip"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  allocation_method   = "Static"
}

resource "azurerm_network_interface" "vm_nic" {
  name                = "${var.prefix}-vm-nic"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  ip_configuration {
    name                          = "primary"
    subnet_id                     = azurerm_subnet.internal.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.pip.id
  }
}

resource "azurerm_network_security_group" "ssh" {
  name                = "ssh"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    access                     = "Allow"
    direction                  = "Inbound"
    name                       = "ssh"
    priority                   = 100
    protocol                   = "Tcp"
    source_port_range          = "*"
    source_address_prefix      = "*"
    destination_port_range     = "22"
    destination_address_prefix = "*"
  }
}

resource "azurerm_network_interface_security_group_association" "vm" {
  network_interface_id      = azurerm_network_interface.vm_nic.id
  network_security_group_id = azurerm_network_security_group.ssh.id

  depends_on = [azurerm_linux_virtual_machine.vm]
}

resource "azurerm_linux_virtual_machine" "vm" {
  name                = "${var.prefix}-vm"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_B1s"
  admin_username      = "micro"

  network_interface_ids = [
    azurerm_network_interface.vm_nic.id
  ]

  admin_ssh_key {
    username   = "micro"
    public_key = file("C:\\Users\\alexa\\.ssh\\id_rsa.pub")
  }

  custom_data = base64encode(file("C:\\Users\\alexa\\Documents\\CoursEfrei\\Cloud\\terraform3\\cloud-init.txt"))

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }

  os_disk {
    storage_account_type = "Standard_LRS"
    caching              = "ReadWrite"
  }
}

```
`cloud-init.txt`
```
#cloud-config
users:
  - default
  - name: micro
    sudo: sudo
    groups: sudo, docker
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-rsa <...>

package_update: true
package_upgrade: true

groups:
  - docker

write_files:
  - path: /usr/share/keyrings/docker.asc
    owner: root:root
    permissions: '0644'
    content: |
      -----BEGIN PGP PUBLIC KEY BLOCK-----

      ....
      -----END PGP PUBLIC KEY BLOCK-----

apt:
  sources:
    docker.list:
      source: deb [arch=amd64 signed-by=/usr/share/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $RELEASE stable

packages:
  - docker-ce
  - docker-ce-cli
  - containerd.io
  - docker-compose-plugin

runcmd:
  - sudo docker pull alpine:latest
```

🌞 **Proof !**
```
PS C:\Users\alexa\Documents\CoursEfrei\Cloud\terraform3> ssh user@52.148.202.135
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 6.8.0-1021-azure x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sun Mar 23 11:05:45 UTC 2025

  System load:  0.71              Processes:             120
  Usage of /:   8.1% of 28.89GB   Users logged in:       0
  Memory usage: 36%               IPv4 address for eth0: 10.0.1.4
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

New release '24.04.2 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Sun Mar 23 11:03:59 2025 from 91.169.101.209
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

user@partie3C-vm:~$ docker images
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
alpine       latest    aded1e1a5b37   5 weeks ago   7.83MB
user@partie3C-vm:~$
```

### B. Go further

`main.tf`
```
provider "azurerm" {
  features {}
  subscription_id = "074aa5b4-e4ab-42f9-807d-4ee508b26e98"
}

resource "azurerm_resource_group" "main" {
  name     = "${var.prefix}-resources"
  location = var.location
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
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_public_ip" "pip" {
  name                = "${var.prefix}-pip"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  allocation_method   = "Static"
}

resource "azurerm_network_interface" "vm_nic" {
  name                = "${var.prefix}-vm-nic"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  ip_configuration {
    name                          = "primary"
    subnet_id                     = azurerm_subnet.internal.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.pip.id
  }
}

resource "azurerm_network_security_group" "ssh" {
  name                = "ssh"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    access                     = "Allow"
    direction                  = "Inbound"
    name                       = "ssh"
    priority                   = 100
    protocol                   = "Tcp"
    source_port_range          = "*"
    source_address_prefix      = "*"
    destination_port_range     = "22"
    destination_address_prefix = "*"
  }

  security_rule {
    access                     = "Allow"
    direction                  = "Inbound"
    name                       = "AllowWikiJS"
    priority                   = 110
    protocol                   = "Tcp"
    source_port_range          = "*"
    source_address_prefix      = "*"
    destination_port_range     = "10101"
    destination_address_prefix = "*"
  }
}

resource "azurerm_network_interface_security_group_association" "pow" {
  network_interface_id      = azurerm_network_interface.vm_nic.id
  network_security_group_id = azurerm_network_security_group.ssh.id
}

resource "azurerm_linux_virtual_machine" "wikijs" {
  name                = "${var.prefix}-vm"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_B1s"
  admin_username      = "micro"

  network_interface_ids = [
    azurerm_network_interface.vm_nic.id
  ]

  admin_ssh_key {
    username   = "micro"
    public_key = file("C:\\Users\\alexa\\.ssh\\id_rsa.pub")
  }

  custom_data = base64encode(file("C:\\Users\\alexa\\Documents\\CoursEfrei\\Cloud\\terraform4\\cloud-init.txt"))

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }

  os_disk {
    storage_account_type = "Standard_LRS"
    caching              = "ReadWrite"
  }
}

```
`cloud-init.txt`
```
#cloud-config
users:
  - default
  - name: user
    sudo: sudo
    groups: sudo, docker
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-rsa <...>

package_update: true
package_upgrade: true

groups:
  - docker

write_files:
  - path: /usr/share/keyrings/docker.asc
    owner: root:root
    permissions: '0644'
    content: |
      -----BEGIN PGP PUBLIC KEY BLOCK-----

      ...
      -----END PGP PUBLIC KEY BLOCK-----
  - path: /opt/wikijs/docker-compose.yml
    owner: root:root
    permissions: '0644'
    content: |
      version: "3.8"

      services:
        db:
          image: mysql:5.7
          restart: always
          ports:
            - '3306:3306'
          volumes:
            - "./db/mysql_files:/var/lib/mysql"
          environment:
            MYSQL_ROOT_PASSWORD: mysql
            MYSQL_DATABASE: wiki
            MYSQL_USER: wiki
            MYSQL_PASSWORD: mysql

        wiki:
          image: requarks/wiki
          restart: always
          depends_on:
            - db
          ports:
            - "10101:3000"
          environment:
            DB_TYPE: mysql
            DB_HOST: db
            DB_PORT: 3306
            DB_USER: wiki
            DB_PASS: mysql
            DB_NAME: wiki

apt:
  sources:
    docker.list:
      source: deb [arch=amd64 signed-by=/usr/share/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $RELEASE stable

packages:
  - docker-ce
  - docker-ce-cli
  - containerd.io
  - docker-compose-plugin

runcmd:
  - docker compose -f /opt/wikijs/docker-compose.yml up -d
```
variable.tf
```
variable "prefix" {
  description = "Project prefix"
  default     = "partie3D"
}

variable "location" {
  description = "Azure region"
  default     = "West Europe"
}
```  
🌞 **Proof !**
```
micro@ MINGW64 ~
$ curl http://52.232.41.218:10101 | tail -n2
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3017  100  3017    0     0  20774      0 --:--:-- --:--:-- --:--:-- 20806
</script><link type="text/css" rel="stylesheet" href="/_assets/css/app.0e32c5b5f7b7df293dc5.css"><script type="text/javascript" src="/_assets/js/runtime.js?1738531300"></script><script type="text/javascript" src="/_assets/js/app.js?1738531300"></script></head><body><div id="root"><page locale="en" path="home" title="j'ai fini" description="" :tags="[]" created-at="2025-03-23T11:51:52.054Z" updated-at="2025-03-23T11:51:52.054Z" author-name="Administrator" :author-id="1" editor="ckeditor" :is-published="true" toc="W10=" :page-id="1" sidebar="W3siaSI6InNkaS0xIiwiayI6ImxpbmsiLCJsIjoiSG9tZSIsImMiOiJtZGktaG9tZSIsInkiOiJob21lIiwidCI6Ii8ifV0=" nav-mode="MIXED" comments-enabled effective-permissions="eyJjb21tZW50cyI6eyJyZWFkIjp0cnVlLCJ3cml0ZSI6ZmFsc2UsIm1hbmFnZSI6ZmFsc2V9LCJoaXN0b3J5Ijp7InJlYWQiOmZhbHNlfSwic291cmNlIjp7InJlYWQiOmZhbHNlfSwicGFnZXMiOnsicmVhZCI6dHJ1ZSwid3JpdGUiOmZhbHNlLCJtYW5hZ2UiOmZhbHNlLCJkZWxldGUiOmZhbHNlLCJzY3JpcHQiOmZhbHNlLCJzdHlsZSI6ZmFsc2V9LCJzeXN0ZW0iOnsibWFuYWdlIjpmYWxzZX19" edit-shortcuts="eyJlZGl0RmFiIjp0cnVlLCJlZGl0TWVudUJhciI6ZmFsc2UsImVkaXRNZW51QnRuIjp0cnVlLCJlZGl0TWVudUV4dGVybmFsQnRuIjp0cnVlLCJlZGl0TWVudUV4dGVybmFsTmFtZSI6IkdpdEh1YiIsImVkaXRNZW51RXh0ZXJuYWxJY29uIjoibWRpLWdpdGh1YiIsImVkaXRNZW51RXh0ZXJuYWxVcmwiOiJodHRwczovL2dpdGh1Yi5jb20vb3JnL3JlcG8vYmxvYi9tYWluL3tmaWxlbmFtZX0ifQ==" filename="home.html"><template slot="contents"><div><p>tp fini&nbsp;</p>
</div></template><template slot="comments"><div><comments></comments></div></template></page></div></body></html>

```
