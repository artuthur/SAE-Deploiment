# SAÉ 3.03 - Déploiement d’une application - Procédure 02

Arthur DEBACQ - Pierre FOULON  
S3-G BUT Informatique

***Semaine 2 - Fin de configuration et mise en place d’un premier service***

---

Table des matières :

- [SAÉ 3.03 - Déploiement d’une application - Procédure 02](#saé-303---déploiement-dune-application---procédure-02)
  - [***Dernière configurations sur la VM***](#dernière-configurations-sur-la-vm)
    - [***Changement du nom de machine***](#changement-du-nom-de-machine)
    - [***Installation et configuration de la commande sudo***](#installation-et-configuration-de-la-commande-sudo)
    - [***Configuration de la synchronisation d’horloge***](#configuration-de-la-synchronisation-dhorloge)
  - [***Installation et configuration basique d’un serveur de base de données***](#installation-et-configuration-basique-dun-serveur-de-base-de-données)
    - [***Installation de la base PostgreSQL***](#installation-de-la-base-postgresql)

---

## ***Dernière configurations sur la VM***

### ***Changement du nom de machine***

Avant changement, votre prompt est:

```bash
user@debian:~$ 
```

Pour changer le nom de la machine, il faut modifier le **hostname**.  Cette variable est stockée dans le fichier `/etc/hostname`

```bash
# nano /etc/hostname
```

Remplacer le nom actuel de la VM par le nom choisi pour la machine.
Après changement, votre prompt est:

```bash
user@nomMachine:~$ 
```

Afin d'avoir une adresse ip associée à notre machine il faut configurer le fichier `/etc/hosts`.
Il s'agit d'un simple fichier texte qui associe des adresses IP avec des noms d'hôtes, une ligne par adresse IP.

```bash
# nano /etc/hosts
```

Remplacer le nom actuel de la VM par le nom choisi pour la machine.

---

### ***Installation et configuration de la commande sudo***

Pour les opérations suivantes, il est nécessaire d'avoir l'accès en root. Pour cela entrer la commande `su -`.
Avant de pouvoir utiliser la commande sudo, il faut l'installer via la commande :

```bash
# apt install sudo
```

Pour ajouter `user` à la liste de sudoers, une commande est possible :

```bash
# usermod -aG sudo user
```

Ici le `-aG` ajoute `user` aux groupes qui suivent l'option, redemarrer la machine.

Attention les variables d'environnement de `root` ne sont pas transmises aux sudoers, on peut donc pas faire de `apt install/update/upgrade`. Pour avoir les mêmes variables il faut éditer le fichier visudo:

```bash
$ sudo visudo
```

Ajouter la ligne qui suit en dessous des variables `Defaults` :

```bash
Defaults env_keep += "ftp_proxy http_proxy https_proxy no_proxy"
```

Sauvegarder et relancer la connection ssh.

---

### ***Configuration de la synchronisation d’horloge***

Pour afficher les événnements systèmes correspondant au *unit service*, on utilise la commande

```bash
$ sudo journalctl -u systemd-timesyncd
```

On peut voir qu'il n'y a pas de synchronisation de l'heure ni de la date. Les tentatives de connection aux serveurs NTP de Debian sont timeout. Le problème vient du proxy de l'université.
Pour palier ce soucis il faut modifier le fichier de configuration avec la commande :

```bash
$ sudo nano /etc/systemd/timesyncd.conf
```

Dans le fichier, décommenter et modifier les lignes suivantes :

```
[Time]
    NTP=
    FallbackNTP=ntp.univ-lille.fr
```

Ensuite il faut redémarrer le service avec la commande :

```bash
$ sudo systemctl restart systemd-timesyncd.service 
```

La commande `date` renvoie maintenant la date et l'heure correcte.

---

## ***Installation et configuration basique d’un serveur de base de données***

### ***Installation de la base PostgreSQL***

- Pour installer postgresql sur notre machine il faut commencer par entrer la commande :

```bash
$ sudo apt install postgresql
```

- Pour vérifier que le service est correctement démarré on utilise :

```bash
$ systemctl status postgresql
```

- On veut créer un utilisateur avec le nom "matrix", pour cela nous devons nous identifier en tant qu'utilisateur postgres avec la commande :

```bash
$ sudo -u postgres -i
```

- Ensuite nous pouvons créer notre utilisateur et son mot de passe "matrix" avec la commande :

```bash
$ createuser matrix -P
```

- Encore avec les permissions de l'utilisateur postgres nous pouvons également créer notre base de donnée avec la commande :

```bash
$ createdb -O matrix matrix
```

> A partir de maintenant notre base de données et notre utilisateur sont créés nous pouvons repasser en user avec la commande `exit`

- Avant de pouvoir utiliser la commande `psql`, il faut changer le mode d'authentification des utilisateurs locaux de psql. Dans le fichier suivant :

```bash
sudo nano /etc/postgresql/13/main/pg_hba.conf
```

- On remplace "peer" par "md5", on obtient donc ceci :

```bash
# "local" is for Unix domain socket connections only
local   all             all                                     md5
```

- Ensuite on redemarre le service psql pour mettre à jour les changements avec la commande :
  
```bash
sudo systemctl restart postgresql
```

On peut désormais executer des ordres SQL avec la commande :

 ```bash
psql -U matrix -c 'CREATE TABLE Test(texte TEXT)'
```

 On peut également afficher ou ajouter des données avec les requêtes INSERT et SELECT, par exemple :

```bash
$ psql -U matrix -c "INSERT INTO Test VALUES('Toto');"
$ psql -U matrix -c "INSERT INTO Test VALUES('Test02');"
$ psql -U matrix -c "SELECT * FROM Test;"
```
