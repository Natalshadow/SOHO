# Simplon - WildCodeSchool - Test positionnement

## Brief part 4 - Installation GLPI sur Debian 11.6

### Préparation et installation du système

Créer une VM en Debian 11.6 pour y installer GLPI. 

| CPU | RAM | Stockage |
|-----|-----|----------|
|  1  | 2 Go|  20 Go   |





Définir le nom sur srv-glpi et le domaine sur rue25.com.
Après création du compte admin, créer la partition disque.

Décocher le Destktop Environment et rester en Headless puis activer le SSH conformément aux consignes du brief et afin d’éviter une reconfiguration post-installation.

![Image](assets/20260118162825.png)


L'os démarre après l'utilitaire d'installation.
L'adresse IP attribuée est le 192.168.1.12.

### Configuration initiale système
Vérifier si SSH s'est mis en marche correctement, cela permet de poursuivre le travail depuis un poste remote.


![Image](assets/20260118163159.png)
![Image](assets/20260118163658.png)

SSH fonctionnel, prouvé par la possibilité de s'y connecter depuis un autre PC.

Installer sudo via su- puis ajouter l'utilisateur au groupe sudo. Nécessaire pour l'installation des paquets.
Ensuite se référer à la documentation officielle de GLPI ici :

- https://glpi-install.readthedocs.io/en/latest/prerequisites.html
- https://glpi-install.readthedocs.io/en/latest/install/index.html


### Installation des dependencies
Tout d'abord, installer les paquets  nécessaires conformément au paragraphe "prerequisites" et "web server" de la documentation :
```bash
sudo apt install apache2 mariadb-server php php-mysql php-ldap php-xml php-gd php-mbstring php-curl php-intl php-zip php-bz2
```
### Configuration database
Initialiser la base de données avec le compte administrateur dédié pour éviter l'accès root.
```bash
sudo mariadb
CREATE DATABASE glpi CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'glpi_admin'@'localhost' IDENTIFIED BY 'unmotdepassetresfort';
GRANT ALL PRIVILEGES ON glpi.* TO 'glpi_admin'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Installation GLPI 
Ensuite, télécharger l'archive GLPI avec :

```bash
wget  https://github.com/glpi-project/glpi/releases/download/10.0.12/glpi-10.0.12.tgz
```

En cas de doute, utiliser "man tar" pour identifier la bonne commande d'extraction puis :
```bash
tar -xvzf glpi-10.0.12.tgz 
```

Résumé de la structure :

|Instance|Configuration|Data|Logs|
|--------|-------------|----|----|
|/var/www/glpi|/etc/glpi|/var/lib/glpi/files|/var/log/glpi|

Déplacer le fichier : 
```bash
sudo mv glpi /var/www/glpi
```

Donner les permissions récursives:
```bash
sudo chown -R www-data:www-data /var/www/glpi 
sudo chmod -R 755 /var/www/glpi
```


Ranger les fichiers aux emplacements indiqués dans le tableau ci-dessus :
```bash
sudo cp -rp /var/www/glpi/config /etc/glpi/
sudo cp -rp /var/www/glpi/files /var/lib/glpi/files/
sudo nano /var/www/glpi/inc/downstream.php
```
Créer unfichier downstream.php qui contient :
```php
<?php
define('GLPI_CONFIG_DIR', '/etc/glpi/');

if (file_exists(GLPI_CONFIG_DIR . '/local_define.php')) {
   require_once GLPI_CONFIG_DIR . '/local_define.php';
}
```

Puis créer le fichier /etc/glpi/local_define.php qui contient :
```php
<?php
define('GLPI_VAR_DIR', '/var/lib/glpi/files');
define('GLPI_LOG_DIR', '/var/log/glpi');
```

### Paramétrage GUI de GLPI
Dorénavant l'interface web est accessible à l'adresse :
http://192.168.1.12/glpi/install/install.php
![Image](assets/20260118165219.png)


Accepter les ToS.
Choisir "Installer"
L'utilitaire signale plusieurs problèmes à corriger :
![Image](assets/20260118165549.png)
Un avertissement concernant la version de PHP est signalé. Le choix a été fait de conserver la version fournie par Debian afin de bénéficier des correctifs de sécurité du dépôt officiel. Ce choix est considéré compatible avec le bon fonctionnement de GLPI dans ce contexte.

Remplir ensuite le formulaire de connexion avec les infos placées dans la commande mariadb plus tôt.
![Image](assets/20260118180550.png)
Connexion établie.

![Image](assets/20260118180628.png)
On sélectionne la base de données existante : glpi


Tester la connexion avec le compte admin:
![Image](assets/20260118180853.png)
Réussi, on a bien tous les comptés créés par défaut :
![Image](assets/20260118180949.png)
A ce stade, il reste à créer les vrais comptes, bloquer, modifier ou supprimer les comptes par défaut inutilisés pour ne pas avoir de faille évidente. Supprimer les fichiers d'installation qui ont été utilisés pour garder un environnement propre pour se simplifier la vie lors de potentiels diagnostiques futurs et être sûr de ne pas laisser la possibilité de déclencher une installation malencontreuse et casser celle en place.

### Pour aller plus loin

Petite note bonus, selon l'environnement, je sais qu'il existe une version Docker de GLPI qui pourrait s'avérer ou non pertinnente. Le docker-compose.yaml ressemblerait à :

![Image](assets/20260118182627.png)
Source :
https://help.glpi-project.org/tutorials/fr/procedures/running_glpi_on_docker
Il suffirait alors de créer le fichier .env, adapter les bind mounts et ajouter un service pour un reverse proxy (nginx par exemple) ou ajouter un tunnel Cloudflared pour bénéficier des protections de Cloudflared.
