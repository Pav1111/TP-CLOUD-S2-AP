# TP1 : Containers

## Part I : Docker basics

## 1. Install

🌞 **Installer Docker votre machine Azure**

- en suivant [la doc officielle](https://docs.docker.com/engine/install/)

```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```


- démarrer le service `docker` avec une commande `systemctl`

``sudo systemctl start docker``

- ajouter votre utilisateur au groupe `docker`
  - cela permet d'utiliser Docker sans avoir besoin de l'identité de `root`
  - avec la commande : `sudo usermod -aG docker $(whoami)`
  - déconnectez-vous puis relancez une session pour que le changement prenne effet

``docker run hello-world`` --> ``Hello from Docker!``


> N'oubliez pas que je veux **toutes** les commandes dans le compte-rendu dès qu'il y a un p'tit 🌞.

## 2. Vérifier l'install

🌞 **Utiliser la commande `docker run`**

- lancer un conteneur `nginx`
  - conf par défaut étou étou, simple pour le moment
  - par défaut il écoute sur le port 80 et propose une page d'accueil
- le conteneur doit être lancé avec un partage de port
  - le port 9999 de la machine hôte doit rediriger vers le port 80 du conteneur

``docker run --name web -d -p 8888:80 nginx``

### Ajouter la bonne règle pour le bon port.


🌞 **Rendre le service dispo sur internet**

- il faut peut-être ouvrir un port firewall dans votre VM (suivant votre OS, ptet y'en a un, ptet pas)
- il faut ouvrir un port dans l'interface web de Azure (appelez moi si vous trouvez pas)
- vous devez pouvoir le visiter avec votre navigateur (un `curl` m'ira bien pour le compte-rendu)

``curl http://98.66.169.145:9999``



🌞 **Custom un peu le lancement du conteneur**

- l'app NGINX doit avoir un fichier de conf personnalisé pour écouter sur le port 7777 (pas le port 80 par défaut)
- l'app NGINX doit servir un fichier `index.html` personnalisé (pas le site par défaut)
- l'application doit être joignable grâce à un partage de ports (vers le port 7777)
- vous limiterez l'utilisation de la RAM du conteneur à 512M
- le conteneur devra avoir un nom : `meow`

``mkdir -p ~/nginx_custom/conf.d``

``mkdir -p ~/nginx_custom/html``

``nano ~/nginx_custom/conf.d/custom_nginx.conf``

```
server {
  listen 7777;
  root /var/www/tp_docker;
  index index.html;
}
````

``nano ~/nginx_custom/html/index.html``

```
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mon Nginx Personnalisé</title>
</head>
<body>
    <h1>Bienvenue sur mon serveur NGINX personnalisé 🎉</h1>
</body>
</html>
```

```
docker run --name meow -d   -p 7777:7777   -v ~/nginx_custom/conf.d/custom_nginx.conf:/etc/nginx/conf.d/custom_nginx.conf   -v ~/nginx_custom/html:/var/www/tp_docker   --memory=512m   nginx
```
🌞 **Call me**

- appelez-moi que je visite votre site web please
- envoyez moi l'IP publique par MP Discord (hésitez pas à me faire signe si je suis en train de voguer entre les tables)

# Part II : Images

## Construisez votre propre Dockerfile

🌞 **Construire votre propre image**

- image de base (celle que vous voulez : debian, alpine, ubuntu, etc.)
  - une image du Docker Hub
  - qui ne porte aucune application par défaut
- vous ajouterez
  - mise à jour du système
  - installation de Apache (pour les systèmes debian, le serveur Web apache s'appelle `apache2` et non pas `httpd` comme sur Rocky)
  - page d'accueil Apache HTML personnalisée

```
mkdir ~/apache_docker && cd ~/apache_docker
touch Dockerfile apache2.conf index.html
```

Dans 'apache2.conf'
```
# on définit un port sur lequel écouter
Listen 80

# on charge certains modules Apache strictement nécessaires à son bon fonctionnement
LoadModule mpm_event_module "/usr/lib/apache2/modules/mod_mpm_event.so"
LoadModule dir_module "/usr/lib/apache2/modules/mod_dir.so"
LoadModule authz_core_module "/usr/lib/apache2/modules/mod_authz_core.so"

# on indique le nom du fichier HTML à charger par défaut
DirectoryIndex index.html
# on indique le chemin où se trouve notre site
DocumentRoot "/var/www/html/"

# quelques paramètres pour les logs
ErrorLog "/var/log/apache2/error.log"
LogLevel warn
````

Dans 'index.html':
```
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mon Apache Custom</title>
</head>
<body>
    <h1>Bienvenue sur mon serveur Apache personnalisé 🎉</h1>
</body>
</html>
```

Dans 'Dockerfile' : 

```
# Utiliser Debian comme image de base
FROM debian:latest

# Mettre à jour le système et installer Apache
RUN apt update -y && apt install -y apache2

# Copier le fichier de configuration Apache personnalisé
COPY apache2.conf /etc/apache2/apache2.conf

# Copier la page web personnalisée
COPY index.html /var/www/html/index.html

# Exposer le port 80 à l'intérieur du conteneur
EXPOSE 80

# Lancer Apache en premier plan
CMD ["apache2", "-DFOREGROUND"]
```

``docker build . -t apache_custom``

``docker run --name my_apache -d -p 8080:80 apache_custom``


# Part III : `docker-compose`

🌞 **Installez un WikiJS** en utilisant Docker

- WikiJS a besoin d'une base de données pour fonctionner
- il faudra donc deux conteneurs : un pour WikiJS et un pour la base de données
- référez-vous à la doc officielle de WikiJS, c'est tout guidé

🌞 **Call me** when it's done

- je dois pouvoir visiter votre WikiJS (il doit être dispo sur internet)


`mkdir ~/wikijs && cd ~/wikijs`

`nano docker-compose.yml`
```
dedans : 
version: "3.8"

services:
  db:
    image: postgres:13
    restart: always
    environment:
      POSTGRES_DB: wikijs
      POSTGRES_USER: wikijs
      POSTGRES_PASSWORD: mypassword
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  wikijs:
    image: requarks/wiki:latest
    restart: always
    depends_on:
      - db
    environment:
      DB_TYPE: postgres
      DB_HOST: db
      DB_PORT: 5432
      DB_USER: wikijs
      DB_PASS: mypassword
      DB_NAME: wikijs
    ports:
      - "8080:3000"

volumes:
  pgdata:
```

`docker compose up -d`

## 3. Make your own meow

🌞 **Vous devez :**

- construire une image qui
  - contient `python3`
  - contient l'application et ses dépendances
  - lance l'application au démarrage du conteneur
- écrire un `docker-compose.yml` qui définit le lancement de deux conteneurs :
  - l'app python
  - le Redis dont il a besoin

`mkdir ~/meow_app && cd ~/meow_app`

Dans app.py: 

```
import redis
import time
import socket
from flask import Flask, request, render_template

# Attente pour s'assurer que Redis démarre
time.sleep(5)

# Connexion à Redis
r = redis.StrictRedis(host='db', port=6379, db=0)

app = Flask(__name__)

@app.route('/')
@app.route('/index')
def index():
    hostname = socket.gethostname()
    return render_template('index.html',
                           title='Home',
                           container_hostname=hostname)

@app.route('/add', methods=['POST', 'GET'])
def add():
    if request.method == 'POST':
        r.set(request.form['key'], request.form['value'])
    return 'Successfully added key ' + request.form['key']

@app.route('/get', methods=['POST'])
def get():
    try:
        if request.method == 'POST':
            keyBytes = r.get(request.form['key'])
            key = keyBytes.decode('utf-8')
        return 'You asked about key ' + request.form['key'] + ". Value : " + key
    except:
        return 'Key ' + request.form['key'] + " does not exist."

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8888)
```

Dans requirements.txt : 

```
flask
redis
````

`mkdir templates && nano templates/index.html`

````
<h1>Add key</h1>
<form action="{{ url_for('add') }}" method="POST">
    Key: <input type="text" name="key">
    Value: <input type="text" name="value">
    <input type="submit" value="Submit">
</form>

<h1>Check key</h1>
<form action="{{ url_for('get') }}" method="POST">
    Key: <input type="text" name="key">
    <input type="submit" value="Submit">
</form>

Host : {{ container_hostname }}

````
Dans Dockerfile : 

```
# Utiliser Python comme image de base
FROM python:3.9

# Définir le dossier de travail
WORKDIR /app

# Copier les fichiers de l'application
COPY . /app

# Installer les dépendances
RUN pip install --no-cache-dir -r requirements.txt

# Exposer le port 8888
EXPOSE 8888

# Lancer l'application Flask
CMD ["python", "app.py"]
```

Dans docker-compose.yml : 
````
version: "3.8"

services:
  db:
    image: redis:latest
    restart: always
    ports:
      - "6379:6379"

  app:
    build: .
    restart: always
    depends_on:
      - db
    ports:
      - "8888:8888"
    volumes:
      - .:/app
````

`docker compose up -d --build` 

`http://98.66.169.145:8888/`

Ne pas oublier de rajouter la règle `6379` (tcp)

# Part IV : Docker security

🌞 **Prouvez que vous pouvez devenir `root`**

- en étant membre du groupe `docker`
- sans taper aucune commande `sudo` ou `su` ou ce genre de choses
- normalement, une seule commande `docker run` suffit
- pour prouver que vous êtes `root`, plein de moyens possibles
  - par exemple un `cat /etc/shadow` qui contient les hash des mots de passe de la machine hôte
  - normalement, seul `root` peut le faire

`groups`

`docker run --rm -v /:/mnt alpine cat /mnt/etc/shadow`

🌞 **Utilisez Trivy**

- effectuez un scan de vulnérabilités sur des images précédemment mises en oeuvre :
  - celle de WikiJS que vous avez build
  - celle de sa base de données
  - l'image de Apache que vous avez build
  - l'image de NGINX officielle utilisée dans la première partie

`sudo apt install -y curl`

`curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh`

`trivy image requarks/wiki:latest`

`trivy image postgres:13`

`trivy image apache_custom`

`trivy image nginx:latest`

🌞 **Utilisez l'outil Docker Bench for Security**

`sudo sh docker-bench-security.sh`

```
117 vérifications effectuées (Checks: 117)
Score de sécurité : 5 (Score: 5)
```
