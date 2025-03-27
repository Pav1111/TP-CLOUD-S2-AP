# TP3 : Automatisation et gestion de conf

## 1. Mise en place

### A. Setup Azure

➜ **Préparez un plan Terraform `main.tf`** (dans un nouveau répertoire de travail dédié à ce TP)

- il crée 2 VMs
- chacune doit être joignable en SSH depuis votre poste
- doit utiliser une image proposée par Azure un minimum à jour
- ajouter une conf `cloud-init.txt` :
  - Ansible et Python installés (référez-vous à la doc d'install de Ansible pour votre l'OS que vous avez choisi)
  - création d'un user qui a accès aux droits `root` avec la commande `sudo`
  - je vous conseille une conf en `NOPASSWD` pour ne pas avoir à saisir votre password à chaque déploiement
  - ce user a une clé publique pour vous y connecter sans mot de passe

🌞 **Vous me livrerez vos deux fichiers en compte-rendu**

- `main.tf` (et éventuellement d'autres fichiers `.tf`)
- `cloud-init.txt`

Dans `main.tf` : 

```
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>3.0"
    }
  }
}

provider "azurerm" {
  features {}
  subscription_id = "d58a36c9-ef32-4457-b16c-48a648b2e175"
}

resource "azurerm_resource_group" "tp3" {
  name     = "tp3-ansible-resources"
  location = "West Europe"
}

resource "azurerm_virtual_network" "tp3_vnet" {
  name                = "tp3-vnet"
  address_space       = ["10.3.1.0/24"]
  location            = azurerm_resource_group.tp3.location
  resource_group_name = azurerm_resource_group.tp3.name
}

resource "azurerm_subnet" "tp3_subnet" {
  name                 = "tp3-subnet"
  resource_group_name  = azurerm_resource_group.tp3.name
  virtual_network_name = azurerm_virtual_network.tp3_vnet.name
  address_prefixes     = ["10.3.1.0/28"]
}

resource "azurerm_public_ip" "vm1_pip" {
  name                = "vm1-public-ip"
  resource_group_name = azurerm_resource_group.tp3.name
  location            = azurerm_resource_group.tp3.location
  allocation_method   = "Static"
}

resource "azurerm_network_interface" "vm1_nic" {
  name                = "vm1-nic"
  resource_group_name = azurerm_resource_group.tp3.name
  location            = azurerm_resource_group.tp3.location

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.tp3_subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.vm1_pip.id
  }
}

resource "azurerm_network_interface" "vm2_nic" {
  name                = "vm2-nic"
  resource_group_name = azurerm_resource_group.tp3.name
  location            = azurerm_resource_group.tp3.location

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.tp3_subnet.id
    private_ip_address_allocation = "Dynamic"
  }
}

resource "azurerm_linux_virtual_machine" "vm1" {
  name                  = "vm1"
  resource_group_name   = azurerm_resource_group.tp3.name
  location              = azurerm_resource_group.tp3.location
  size                  = "Standard_B1s"
  admin_username        = "micro"
  network_interface_ids = [azurerm_network_interface.vm1_nic.id]

  admin_ssh_key {
    username   = "micro"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  custom_data = base64encode(file("cloud-init.txt"))

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

resource "azurerm_linux_virtual_machine" "vm2" {
  name                  = "vm2"
  resource_group_name   = azurerm_resource_group.tp3.name
  location              = azurerm_resource_group.tp3.location
  size                  = "Standard_B1s"
  admin_username        = "micro"
  network_interface_ids = [azurerm_network_interface.vm2_nic.id]

  admin_ssh_key {
    username   = "micro"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  custom_data = base64encode(file("cloud-init.txt"))

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

Dans `cloud-init.txt` : 

```
#cloud-config
users:
  - name: micro
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDAzqqtqpmVrP6D2CFB+mml1mvJx1l++3fuQbXQtDmPqzmxdJG1mf6G+K1eM5cuMkwOOL7bLJamouR2yh9++JCxY9s7kZLV6Y2tgAAJOQxAM2QcYgGX9YJIlYAvxzJgV6aFxbkp/yy1nPSNoPasHzVb7UUYHrqB6IcYrhjPycPASsC9AMk2ZtwavHvQNEDI7rZH0Ul2MsPMEf8sww8t+UXuTeSNrpNRevS2VZUz6RmlMJnV2zQKFlJ1vw4kmoeEvxM9oTAF1rKbaR9risDUhAQmfb8VYvASVR1jCs9nLBJhyNiim+z1rwc7BEUCoEVb4CFtHsCs8gdIipepvyumnPtr+42kXV8n2H6bs+HoPiUwElNfrv/hiD8ZmH6mtwLlDmaiQzQgthwL3zxdIZxLQ2VQBefpottEBFAEek68tLBOoz+fJbLv71bHnjKfTP0bhZ9LzQ6TAbhZCzsTZkbkCU7eyFxlzCIdKkWsspTWFIwLaqT8CMi+e1APy4OI4tOU43UWxkoSUi4vUbHkh6izW2nWGWrTlruVG7LgsPDL2fyq6XKFbxtr1ULm5LJjVZrVo8ebLsAjwTWPGnMm5kc1XvKoqhYUVgaWlmUfBG13QolJ2EU/J8UN+TpgxjdZHzxVGwnt+XWgmhdSMIYdtlyvL+QJ12mNJhjNttj3eeUdlbQfkw== alexa@PAVASUS

package_update: true
package_upgrade: true
packages:
  - python3
  - python3-pip
  - ansible

runcmd:
  - systemctl enable ssh
  - systemctl start ssh
```

🌞 **Pour le compte-rendu**

- `nginx.yml` et `hosts.ini` dans le compte-rendu
- un ptit `curl` vers l'interface Web NGINX

`nginx.yml` : 

```
---
- name: Setup NGINX with HTTPS
  hosts: web
  become: true
  tasks:

    - name: Install required packages
      apt:
        name:
          - nginx
          - openssl
        state: present
        update_cache: yes

    - name: Create SSL certificate directory
      file:
        path: /etc/nginx/ssl
        state: directory
        mode: '0755'

    - name: Generate self-signed SSL certificate
      command: >
        openssl req -new -newkey rsa:2048 -days 365 -nodes -x509
        -subj "/CN=localhost" -keyout /etc/nginx/ssl/nginx.key -out /etc/nginx/ssl/nginx.crt
      args:
        creates: /etc/nginx/ssl/nginx.crt

    - name: Create web root directory
      file:
        path: /var/www/tp3_site
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'

    - name: Create a simple index.html file
      copy:
        dest: /var/www/tp3_site/index.html
        content: "<h1>Hello from Ansible with HTTPS!</h1>"
        owner: www-data
        group: www-data
        mode: '0644'

    - name: Deploy NGINX configuration for HTTPS
      copy:
        dest: /etc/nginx/sites-available/tp3_site
        content: |
          server {
              listen 443 ssl;
              server_name _;

              ssl_certificate /etc/nginx/ssl/nginx.crt;
              ssl_certificate_key /etc/nginx/ssl/nginx.key;

              root /var/www/tp3_site;
              index index.html;

              location / {
                  try_files $uri $uri/ =404;
              }
          }
        owner: root
        group: root
        mode: '0644'

    - name: Enable site configuration
      file:
        src: /etc/nginx/sites-available/tp3_site
        dest: /etc/nginx/sites-enabled/tp3_site
        state: link

    - name: Remove default NGINX site
      file:
        path: /etc/nginx/sites-enabled/default
        state: absent

    - name: Restart NGINX
      service:
        name: nginx
        state: restarted
```
`hosts.ini` : 

```
[tp3]
vm1
vm2

[web]
vm1
```

`curl` : 

```
alex_pav@PAVASUS:/mnt/c/Users/alexa/tp3-ansible/ansible$ curl -k https://4.210.156.66
<h1>Hello from Ansible with HTTPS!</h1>alex_pav@PAVASUS:/mnt/c/Users/alexa/tp3-ansible/ansible$
```

### B. MariaDB ou MySQL

➜ **Créez un *playbook* `mariadb.yml`** (ou `mysql.yml`)

- déploie un serveur 
  - MariaDB si vous avez choisi une base RedHat (Rocky, Alma, etc.)
  - ou MySQL si vous utilisez une base Debian  (Debian, Ubuntu, etc.)
- crée un user SQL ainsi qu'une base de données sur laquelle il a tous les droits

➜ **Modifiez votre `hosts.ini`**

- ajoutez une section `db`
- elle ne contient que `10.3.1.12`

➜ **Lancez votre playbook sur le groupe `db`**

➜ **Vérifiez en vous connectant à la base que votre conf a pris effet**

🌞 **Pour le compte-rendu**

- `mariadb.yml` (ou `mysql.yml`) et `hosts.ini` dans le compte-rendu

> N'oubliez pas d'ouvrir le port 3306 avec une règle Azure (modifiez et ré-appliquez votre fichier `main.tf`).

`mariadb.yml` : 

```
---
- name: Install and configure MariaDB
  hosts: db
  become: true

  tasks:
    - name: Install MariaDB server
      apt:
        name: mariadb-server
        state: present

    - name: Start and enable MariaDB service
      service:
        name: mariadb
        state: started
        enabled: true

    - name: Install PyMySQL for Ansible
      apt:
        name: python3-pymysql
        state: present

    - name: Create database
      mysql_db:
        name: mydatabase
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: Create user and grant privileges
      mysql_user:
        name: myuser
        password: mypassword
        priv: 'mydatabase.*:ALL'
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock
```

`hosts.ini` : 

```
[tp3]
vm1
vm2

[web]
vm1

[db]
vm3
```

# III. Repeat

🌞 **En compte-rendu...**

- tous les fichiers modifiés/ajoutés

`inventories/vagrant_lab/host_vars/vm1.yml`: 

```
vhosts:
  - test2:
      nginx_servername: test2
      nginx_port: 8082
      nginx_webroot: /var/www/html/test2
      nginx_index_content: "<h1>teeeeeest 2</h1>"
  - test3:
      nginx_servername: test3
      nginx_port: 8083
      nginx_webroot: /var/www/html/test3
      nginx_index_content: "<h1>teeeeeest 3</h1>"
```

`roles/nginx/tasks/vhosts.yml`:

```
- name: Create webroots
  file:
    path: "{{ item.value.nginx_webroot }}"
    state: directory
    owner: root
    group: root
    mode: '0755'
  loop: "{{ vhosts | dict2items }}"

- name: Create index
  copy:
    dest: "{{ item.value.nginx_webroot }}/index.html"
    content: "{{ item.value.nginx_index_content }}"
    owner: root
    group: root
    mode: '0644'
  loop: "{{ vhosts | dict2items }}"

- name: NGINX Virtual Hosts
  template:
    src: vhost.conf.j2
    dest: "/etc/nginx/conf.d/{{ item.key }}.conf"
  notify: Reload nginx
  loop: "{{ vhosts | dict2items }}"

- name: Open firewall ports for vhosts
  ansible.builtin.command:
    cmd: "ufw allow {{ item.value.nginx_port }}"
  loop: "{{ vhosts | dict2items }}"
  ignore_errors: true
```

`roles/nginx/templates/vhost.conf.j2` : 

```
server {
    listen {{ item.value.nginx_port }};
    server_name {{ item.value.nginx_servername }};

    location / {
        root {{ item.value.nginx_webroot }};
        index index.html;
    }
}
```

`roles/nginx/handlers/main.yml` : 

```
- name: Reload nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```

