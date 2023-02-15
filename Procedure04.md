# SAÉ 3.03 - Déploiement d’une application - Procédure 04

Arthur DEBACQ - Pierre FOULON  
S3-G BUT Informatique

***Semaine 4 - Installation et configuration de Element Web & Reverse Proxy***

---

Table des matières :

- [SAÉ 3.03 - Déploiement d’une application - Procédure 04](#saé-303---déploiement-dune-application---procédure-04)
  - [***Element Web***](#element-web)
    - [***Choix du serveur web***](#choix-du-serveur-web)
    - [***Mise en place de Element***](#mise-en-place-de-element)
  - [***Reverse proxy pour Synapse***](#reverse-proxy-pour-synapse)
    - [***Introduction et choix d’un reverse proxy***](#introduction-et-choix-dun-reverse-proxy)
    - [***Installation d’un reverse proxy***](#installation-dun-reverse-proxy)

---

## ***Element Web***

### ***Choix du serveur web***

> Il y a plusieurs raisons pour lesquelles on avons choisi d'utiliser Nginx plutôt qu'Apache pour un serveur web. Tout d'abord, Nginx est généralement plus rapide et utilise moins de ressources que Apache, ce qui le rend particulièrement adapté pour les serveurs web à haute charge.  
> En outre, Nginx offre une meilleure gestion des connexions simultanées, ce qui peut être un avantage pour les sites web qui reçoivent un grand nombre de visites en même temps comme ici avec Element qui est une plateforme de discussion. Enfin, Nginx est souvent considéré comme plus facile à configurer et à gérer que Apache.
>
> - Nous avons également déjà [installé Nginx](Procedure03.md) et fait [les redirections SSH dans la procédure précédente](Procedure01.md#quelques-trucs-en-plus)

### ***Mise en place de Element***

Premièrement il faut télécharger l'archive de Element, on se place dans le dossier :


```bash
$ cd /var/www/html
```

Puis :

```bash
$ sudo wget https://github.com/vector-im/element-web/releases/download/v1.11.17/element-v1.11.17.tar.gz
```

Pour désarchiver, on utilise :

```bash
$ sudo tar -xf element-v1.11.17.tar.gz
```

Une fois les fichiers décompressés, on les extrait du dossier element-v1.11.17.tar.gz vers le dossier courant.

```bash
$ sudo mv element-v1.11.17/* ./
```

Pour créer le fichier de configuration du site :

```bash
$ sudo cp config.sample.json config.json
```

Modifier la section `m.homserver` du fichier de configuration nouvellement créer `config.json` :

```bash
"m.homeserver": {
            "base_url": "https://virtu.iutinfo.fr:9090/",
            "server_name": "virtu.iutinfo.fr"
        },
```

Donner les droits à root pour tous les fichiers de Element :

```bash
$ sudo chown root:root /var/www/html/ -R
```

---

## ***Reverse proxy pour Synapse***

### ***Introduction et choix d’un reverse proxy***

> Un reverse proxy est un type de serveur qui agit comme un intermédiaire entre les utilisateurs finaux et d'autres serveurs. Lorsqu'un utilisateur envoie une demande à un reverse proxy, celui-ci transmet la demande à un autre serveur en son nom, puis renvoie la réponse de ce serveur à l'utilisateur.
>
>Cela permet au reverse proxy de remplir plusieurs fonctions :
>
> - Améliorer les performances en répartissant la charge entre plusieurs serveurs et en mettant en cache les réponses à certaines requêtes.
> - Sécuriser les communications en agissant comme un pare-feu et en effectuant des opérations de cryptage.
>- Filtrer les demandes en fonction de critères prédéfinis et rediriger les utilisateurs vers des pages spécifiques en fonction de leur identité ou de leur emplacement géographique.
>
> Il existe de nombreux logiciels capables de fonctionner comme reverse proxy, notamment :
>
> - Apache HTTP Server
> - Nginx
> - HAProxy
> - Varnish
> - Squid
>
> Nous avons fait le choix d'utiliser le reverse proxy proposé par Nginx, il est inclut avec le serveur web et la configuration est simple

### ***Installation d’un reverse proxy***

- Nous allons créer une nouvelle machine virtuelle qui va nous servir de reverse proxy. On répete les étapes de la [procédure 1](Procedure01.md) en configurant :
  - Le nom de la machine : `rproxy`
  - l'adresse ip : `192.168.194.4`

- Celle-ci sera accéssible en SSH tout comme notre serveur `Matrix`

Installer le service `nginx` :

```bash
$ sudo apt install nginx
```

- Le reverse proxy écoutera sur le port 80 et redirigera toutes les requetes vers notre serveur matrix.

Modifier le fichier de configuration NGinx :

```bash
$ sudo nano /etc/nginx/sites-available/default 
```

```bash
location / {
    proxy_pass http://192.168.194.3:8008;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;
    client_max_body_size 50M;
}
```

Redémarrer le service Nginx :

```bash
$ sudo systemctl restart nginx
```

> A partir d'ici passer sur la machine `matrix`

Changer le port `server_name: acajou05.iutinfo.fr:8008` en **9090** avec la commande:

```bash
$ sudo nano /etc/matrix-synapse/conf.d/server_name.yaml   
```

Stopper le service matrix :

```bash
$ sudo systemctl stop matrix-synapse.service
```

Passer en mode utilisateur `postgres` pour détruire la table matrix existante. Etant donné qu'on a changé le port, celui-ci est pris en compte lors de la création de la table.

```bash
sudo -u postgres -i
```

Puis pour supprimer la table :

```bash
postgres@matrix:~$ dropdb matrix 
```

Et la recréer avec les paramètres suivants :

```bash
postgres@matrix:~$ createdb --encoding=UTF8 --locale=C --template=template0 --owner=matrix matrix
```

Ajouter l'ip de la machine `matrix` dans le fichier de configuration Matrix-Synapse :

```bash
$ sudo nano /etc/matrix-synapse/homeserver.yaml
```

Y ajouter dans la ligne suivante l'ip de notre machine `192.168.194.3` dans la section `listener`

```bash
bind_addresses: ['::1', '127.0.0.1','192.168.194.3']
```

Redémarrer le serveur Matrix et le reverse proxy avec la commande sur chacune des vm :

```bash
$ sudo reboot
```

Modifier dans le fichier `.ssh/config` afin d'avoir cette configuration dans le fichier pour la connection aux machines matrix (précédemment nommé `vm` puis on supprime les `LocalForward` du host **matrix**) et rproxy :

```bash
Host matrix
    User user
    HostName 192.168.194.3
    ForwardAgent yes

Host rproxy
   User user
   HostName 192.168.194.4
   ForwardAgent yes
   LocalForward 0.0.0.0:9090 192.168.194.4:80
```

Maintenant on accède directement au service Matrix-Synapse en passant par le port **9090** à l'aide du reverse-proxy Nginx.
