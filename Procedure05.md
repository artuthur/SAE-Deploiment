# SAÉ 3.03 - Déploiement d’une application - Procédure 05

Arthur DEBACQ - Pierre FOULON  
S3-G BUT Informatique

***Semaine 5 - Migration vers l’architecture finale***

---

Table des matières :

- [SAÉ 3.03 - Déploiement d’une application - Procédure 05](#saé-303---déploiement-dune-application---procédure-05)
  - [***Architecture finale - Recréation des VM***](#architecture-finale---recréation-des-vm)
    - [***Configuration SSH*** :](#configuration-ssh-)
    - [***Configuration des adresses*** :](#configuration-des-adresses-)
    - [***Configuration du fichier de résolution DNS local*** :](#configuration-du-fichier-de-résolution-dns-local-)
    - [***Configuration du proxy*** :](#configuration-du-proxy-)
    - [***Mise à jour des paquets et installation des services*** :](#mise-à-jour-des-paquets-et-installation-des-services-)
    - [***Configuration du nom de la machine*** :](#configuration-du-nom-de-la-machine-)
    - [***Configuration de la commande sudo*** :](#configuration-de-la-commande-sudo-)
    - [***Synchronisation du temps NTP*** :](#synchronisation-du-temps-ntp-)
  - [***Configuration de notre serveur Web Matrix***](#configuration-de-notre-serveur-web-matrix)
  - [***Configuration de notre Base de données Postgres***](#configuration-de-notre-base-de-données-postgres)
  - [***Configuration de notre Reverse Proxy***](#configuration-de-notre-reverse-proxy)
  - [***Configuration de notre client Element***](#configuration-de-notre-client-element)
  - [***Finalité***](#finalité)


---

## ***Architecture finale - Recréation des VM***

Notre architecture actuelle présente **des failles** car seul le reverse proxy s'éxecute sur une machine virtuelle à part entière. En effet, notre base de données (Postgres), notre client (Element) et notre serveur Web (Matrix) tournent **tous sur une seule machine** ce qui peux présenter des risques :

- Sécurité
- Compatibilité de dépendances
- Dépendances des services à notre unique machine.

Nous allons donc migrer vers une **architecture plus repartie** ce qui nous protégera des failles de sécurité et de la dépendances des autres services à notre unique machine. Pour ce faire nous allons utiliser, comme pour le reverse proxy, **une machine à part entière pour chaque services**.

Nous allons donc dans un premier temps **supprimer** notre unique **machine matrix** contenant l'ensemble des services :

```bash
$ virtvmiut supprimer matrix
```

Maintenant nous allons **créer nos différentes machines virtuelles** qui habriteront nos services :

```bash
$ vmiut creer db
$ vmiut creer element
$ vmiut creer matrix
```

### ***Configuration SSH*** :

Une fois les machines virtuelles crées, pour l'aspect pratique nous allons **modifier le fichier de configuration SSH** afin de **configurer les hôtes** dans l'optique d'utiliser **le nom de la machine virtuelle** au lieu de l'**adresse** de la machine. De plus, nous allons pouvoir configurer le déploiements de nos clés SSH.

Modifier le fichier de configuration SSH **depuis la machine de virtualisation** :

```bash
$ nano .ssh/config
```

Puis y inscrire les host des différentes machines virtuelles :

>Note : nous avons ajoute un LocalForward dans la configuration du host rproxy afin que le reverse proxy soit accessible sur le port 9090.

```bash
Host matrix
    User user
    HostName 192.168.194.3
    ForwardAgent yes

Host rproxy
    User user
    HostName 192.168.194.4
    ForwardAgent yes
    LocalForward 0.0.0.0:9090 192.168.194.4:9090

Host db
    User user
    HostName 192.168.194.5
    ForwardAgent yes

Host element
    User user
    HostName 192.168.194.6
    ForwardAgent yes
```

**<span style="color: red">A partir de cette section il sera nécessaire de repéter ses étapes de configuration pour **CHAQUE** machine virtuelle !**

Nous allons maintenant **configurer** nos diffèrentes machines fraîchement crée comme nous l'avons vu dans les précédentes procédures. Cependant les configurations dispatchées dans les différentes procédures nous allons **généralisé la procédure de configuration** :

Dans un premier temps, nous allons configurer **l'adresse de la machine**,  **depuis la machine de virtualisation** nous pouvons **accéder** à la machine virtuelle souhaité en **utilisant les host** que nous avons créer dans le fichier de configuration SSH.

```bash
$ ssh nomDeLaMachine
```

### ***Configuration des adresses*** :

Une fois la console de la machine virtuelle ouverte, nous allons pouvoir nous connecter en root puis modifier le fichier de configuration des interfaces afin d'y inscrire l'adresse ip que l'on souhaite lui attribué :

```bash
$ su -
```

```bash
# nano /etc/network/interfaces
```

puis modifier **l'interface enp0s3** afin de remplacer les `XX` par l'**adresse de la machine souhaité** et le gateway :  

```bash
iface enp0s3 inet static 
    address 192.168.194.XX
    gateway 192.168.194.2
```
### ***Configuration du fichier de résolution DNS local*** :

Ensuite il faut **configurer le DNS** :

```bash
# nano /etc/resolv.conf
```

Puis inscrire la configuration DNS suivante :

```bash
domain univ-lille.fr
search univ-lille.fr 
nameserver 192.168.194.2
```

### ***Configuration du proxy*** : 

Une fois le DNS configuré, il faut **configurer le proxy** sans quoi **nous ne pourrons pas installer** les paquets et leur mises à jour nécessaires :

```bash
# nano /etc/environnement
```

```bash
HTTP_PROXY=http://cache.univ-lille.fr:3128
HTTPS_PROXY=http://cache.univ-lille.fr:3128
http_proxy=http://cache.univ-lille.fr:3128
https_proxy=http://cache.univ-lille.fr:3128
NO_PROXY=localhost,192.168.194.0/24,172.18.48.0/22
```

Afin de prendre en compte les changements nous nous devons de redémarrer la machine :

```bash
# reboot
```

### ***Mise à jour des paquets et installation des services*** :

Reconnectons-nous à la machine en utilisateur root comme vu précédemment puis il est temps de **mettre à jour les paquets** de la machine :

```bash
# apt update && apt full-upgrade
```

Puis cochez la case [ ] /dev/sda à l'aide de la barre ESPACE lors du processus et valider avec ENTRER.


Pour la machine virtuelle Matrix :
- Installer le service Matrix-Synapse, voir [section correspondante](Procedure03.md#installation-de-synapse)  (se limiter à l'installation du service).

Pour la machine virtuelle DataBase ou db :
- Installer le service Postgres : `sudo apt install postgresql`
    
    Se connecter en tant que postgres :
-   ```bash
    sudo -u postgres -i
    ```
    Créer un utilisateur nommé db :
-   ```bash
    createuser db -P
    ```
    Créer la base de donnée :
-   ```bash
    createdb --encoding=UTF8 --locale=C --template=template0 --owner=db db
    ```

Pour le machine virtuelle Element :
- Installer le service Nginx : `sudo apt install nginx`

- Mise en place d'Element : [Section correspondante](Procedure04.md#mise-en-place-de-element) (ne pas faire le reverseProxy de Synapse)  

Afin de prendre en compte les changements nous nous devons de redémarrer la machine :

```bash
# reboot
```

### ***Configuration du nom de la machine*** :

Une fois ceci fait, reconnetons nous à notre machine et **modifions le nom dans le terminal** pour mieux s'y retrouver pour ceci modifier **le fichier hostname** :

```bash
# nano /etc/hostname
```

Puis inscrire le nom de la machine virtuelle.

Afin d'avoir une **adresse ip associée à notre machine** il faut configurer le fichier `/etc/hosts`. Il s'agit d'un simple fichier texte qui associe des adresses IP avec des noms d'hôtes, une ligne par adresse IP.

```bash
# nano /etc/hosts
```
Remplacer le **nom actuel de la VM** par **le nom choisi pour la machine**.

### ***Configuration de la commande sudo*** :

Pour les opérations suivantes, il est nécessaire d'avoir l'accès en root. Pour cela entrer la commande `su -`. Avant de pouvoir utiliser la commande sudo, il faut l'installer via la commande :

```bash
# apt install sudo
```

Pour ajouter notre utilisateur user à la liste de sudoers :

```bash
# usermod -aG sudo user
```

Redémarrer la machine afin de prendre les changements en compte :

```bash
# reboot
```

Nous allons transmettre les variables d'environnement de root aux sudoers

```bash
$ sudo visudo
```
puis inscrire cette ligne dessous les autres variables d'environnements :

```bash
Defaults env_keep += "ftp_proxy http_proxy https_proxy no_proxy"
```

Redémarrer la machine afin de prendre les changements en compte :

```bash
# reboot
```

### ***Synchronisation du temps NTP*** :

Nous allons maintenant synchronisés l'horloge, modifier le fichier `timesync.conf` :
```bash
$ sudo nano /etc/systemd/timesyncd.conf
```

Dans le fichier, décommenter et modifier les lignes suivantes :

```bash
[Time]
    NTP=
    FallbackNTP=ntp.univ-lille.fr
```

Redémarrer le service :

```bash
$ sudo systemctl restart systemd-timesyncd.service
```

La commande `date` ci-dessous permet de verifier si l'horloge est bien synchronisé :
```bash
$ date
```

**<span style="color: red">Fin des étapes de configuration. À effectuer pour **CHAQUE** machine virtuelle !**

---

## ***Configuration de notre serveur Web Matrix***

Depuis la machine de virtualisation, rendez-vous sur notre machine virutelle `matrix` sur laquelle est installé notre serveur Web Matrix :

```bash
$ ssh matrix
```

Nous devons maintenant configurer notre serveur Web Matrix-Synapse afin qu'il **utilise la base de données postgres** ainsi qu'il **écoute pour les connexions entrantes sur l'adresse 192.168.194.3**.

Tout d'abord nous allons éditer **la variable de configuration** `server_name` :

```bash
$ sudo nano /etc/matrix-synapse/conf.d/server_name.yaml
```

Avec la valeur de configuration suivante afin de **définir le nom d'hôte** que notre serveur utilisera pour **s'identifier lui-même** lorsqu'il **communique** avec d'autres serveurs ou clients : 

```bash
nomdelamachinedevirtualisation.iutinfo.fr:9090
``` 

Nous pouvons dès à present nous attaquer à notre configuration, dans un premier temps nous allons nous rendre dans **notre fichier de configuration** 

```bash
$ sudo nano /etc/matrix-synapse/homeserver.yaml
```

Puis inscrire la variable de configuration suivante :

```bash
bind_addresses: ['192.168.194.3','::1', '127.0.0.1']
```

Toujours dans le même fichier de configuration, nous allons **modifier la section database** afin que Synapse **utilise notre base de données** `db` précedemment créee :

```bash
database:
  name: psycopg2
  args:
    user: db
    password: db
    database: db
    host: 192.168.194.5
    cp_min: 5
    cp_max: 10
```

Nous allons ensuite **créer deux utilisateurs sur notre serveur Web** avec lesquels nous nous connecterons plus tard sur le client Element :

```bash
$ register_new_matrix_user -u pierre -p pierre -c /etc/matrix-synapse/homeserver.yaml
$ register_new_matrix_user -u arthur -p arthur -c /etc/matrix-synapse/homeserver.yaml
```

Nous pouvons maintenant **redémarrer le service Matrix** afin de prendre en compte nos **changements de configurations** :

```bash
$ sudo systemctl restart matrix-synapse.service
```

---

## ***Configuration de notre Base de données Postgres***

Depuis la machine de virtualisation, rendez vous sur notre machine virutelle `db` sur laquelle est installé notre base de données de notre service : 

```bash
$ ssh db
```

Nous devons maintenant **configurer Postgres** afin qu'il **accepte les connexions entrante** depuis notre **serveur Web Matrix** c'est-à-dire notre machine virtuelle Matrix.

Pour ce faire il nous faut nous rendre dans notre **fichier de configuration** Postgres :

```bash
$ sudo nano /etc/postgresql/13/main/postgresql.conf
```

Et y inscrire la variable de configuration et sa valeur attribué ci-dessous :

```bash
listen_addresses = '*'
```

Avec cela nous permettons à **n'importe quel client** de se connecter au serveur, indépendamment de son emplacement car la variable de configuration définit les adresses IP sur lesquelles un serveur doit **écouter pour des connexions entrantes**. La valeur `'*'` signifie que le **serveur écoutera sur toutes les adresses IP disponibles sur l'ordinateur hôte**, y compris les **adresses IP locales** et **publiques**.

Ensuite, nous allons éditer notre **fichier de configuration des regles de sécurité pour les connexions a la base de données** (`pga_hba.conf`) :

```bash
$ sudo nano /etc/postgresql/13/main/pg_hba.conf
```

Pour y renseigner la ligne suivante qui permet d'**autoriser la connexion à la base de données** PostgresSQL **depuis** le réseau 192.168.194.0/24 :

```bash
# db test
host    all             all             192.168.194.3/24        md5
```

Pour prendre en compte le changements, **redémarrer le service postgres** :

```bash
$ sudo systemctl restart postgres.service
```

---

## ***Configuration de notre Reverse Proxy***

Depuis la machine de virtualisation, rendez-vous sur notre machine virutelle `rproxy` contenant notre reverse Proxy :

```bash
$ ssh rproxy
```

Nous allons maintenant configurer notre reverse Proxy afin que :
- l'URL http://virtu.iutinfo.fr:9090 sert notre service Synapse
- l’URL http://virtu.iut-infobio.priv.univ-lille1.fr:9090 sert notre client Element.

Pour ce faire, nous allons **éditer** le fichier nommé `default` dans l'arboresences de notre **service nginx** :

```bash
$ sudo nano /etc/nginx/sites-enables/default
```

Pour y inscrire les configurations suivantes : 

- Dans un premier temps nous allons configurer **l'accès par l'URL** virtu.iutiutinfo.fr:9090 **serve notre service Synapse** à l'adresse http://192.168.194.3:8008, adresse de notre machine virtuelle contenant notre serveur Web `matrix`.

- Puis nous allons configurer **l'accès par l'URL** virtu.iut-infobio.priv.univ-lille1.fr:9090 **serve notre client Element** à l'adresse http://192.168.194.6:80, adresse de notre machine virtuelle contenant notre client `element`.

```
server {
        listen 9090;

        server_name virtu.iutiutinfo.fr;

        location / {
                proxy_pass http://192.168.194.3:8008;
                proxy_set_header X-Forwarded-For $remote_addr;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_set_header Host $host;
                client_max_body_size 50M;
                proxy_http_version 1.1;
        }
}

server {
        listen 9090;

        server_name virtu.iut-infobio.priv.univ-lille1.fr;

        location / {
                proxy_pass http://192.168.194.6:80;
                proxy_set_header X-Forwarded-For $remote_addr;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_set_header Host $host;
                client_max_body_size 50M;
                proxy_http_version 1.1;
        }
}
```

Redémarrer le service nginx afin de prendre en compte les changements pour le reverse Proxy :

```bash
$ sudo systemctl restart nginx.service
```

---

## ***Configuration de notre client Element***

Depuis la machine de virtualisation, rendez-vous sur notre machine virutelle `element` sur laquelle notre client Element est installé :

```bash
$ ssh element
```

Puis nous allons **éditer notre fichier de configuration** `config.json` afin d'**indiquer a notre serveur Matrix l'URL auqiel le client doit se connecter** ainsi que le **nom d'hôte du serveur Matrix auquel le client doit se connecter** :

```bash
$ sudo nano /var/www/html/config.json
```

Puis y renseigner la variable de configuration ci-dessous :

```bash
"m.homeserver": {
    "base_url": "http://virtu.iutinfo.fr:9090/",
    "server_name": "virtu.iutinfo.fr"
},
```

---

## ***Finalité***

Notre achitecture finale étant **installé** et **configuré** nous n'avons plus qu'à **accéder à notre service de disscusion en ligne**, **notre Client Element** en passant par l'URL http://virtu.iut-infobio.priv.univ-lille1.fr:9090/ sur notre Navigateur Web, nous pouvons utilisé le login et mot de passe [**des utilisateurs précédemment créer**](Procedure05.md#configuration-de-notre-serveur-web-matrix) pour nous connecter à notre service de disscusion. 

**/!\ ATTENTION/!\ ,** toute fois à avoir allumer et s'être connecté à la machine virtuelle rproxy sans quoi le reverse proxy ne pourra pas être opérationnel et donc l'accès à notre service impossible !

De même pour l'accès à notre serveur matrix : http://virtu.iutiutinfo.fr:9090/