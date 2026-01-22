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


### Installation des dépendances
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

Donner les permissions récursives :
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
Créer un fichier downstream.php qui contient :
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
Un avertissement concernant la version de PHP est signalé. Le choix a été fait de conserver la version fournie par Debian afin de bénéficier des correctifs de sécurité du dépôt officiel. Ce choix ne nuit pas au bon fonctionnement de GLPI dans ce contexte.

Remplir ensuite le formulaire de connexion avec les identifiants créés dans la commande mariadb plus tôt.
![Image](assets/20260118180550.png)
Connexion établie.

![Image](assets/20260118180628.png)
On sélectionne la base de données existante : glpi


Tester la connexion avec le compte admin :
![Image](assets/20260118180853.png)
Réussi, on a bien tous les comptes créés par défaut :
![Image](assets/20260118180949.png)
Créer les vrais comptes, bloquer, modifier ou supprimer les comptes par défaut inutilisés pour ne pas avoir de faille évidente. Supprimer les fichiers d'installation qui ont été utilisés pour garder un environnement propre pour se simplifier la vie lors de potentiels diagnostiques futurs et être sûr de ne pas laisser la possibilité de déclencher une installation malencontreuse et casser celle en place.


### Mise à jour de GLPI

!!! danger "Important : réaliser une sauvegarde de la base de données."
Télécharger la version à jour ou la version souhaitée via github ou wget.
```bash
wget https://github.com/glpi-project/glpi/releases/download/x.x.x/glpi-x.x.x.tgz
tar -xvzf glpi-x.x.x.tgz
```
Supprimer les data applicatif de l'ancienne version :
```bash
sudo rm -rf /var/www/glpi/* 
sudo cp -r glpi/* /var/www/glpi/
sudo chown -R www-data:www-data /var/www/glpi
```

Recréer le fichier downstream de la même manière qu'à l'installation.
```php
<?php
define('GLPI_CONFIG_DIR', '/etc/glpi/');

if (file_exists(GLPI_CONFIG_DIR . '/local_define.php')) {
   require_once GLPI_CONFIG_DIR . '/local_define.php';
}
```
Lancer ensuite la migration PHP. 
```bash
sudo -u www-data php /var/www/glpi/bin/console db:update
```

Si la version PHP est trop ancienne, une erreur telle que ci-dessous pourrait s'afficher :
```bash
/var/www/glpi$ sudo -u www-data php bin/console db:update
>PHP Parse error:  syntax error, unexpected 'private' (T_PRIVATE), expecting variable (T_VARIABLE) in /var/www/glpi/src/Glpi/Application/ResourcesChecker.php on line 42
```
Auquel cas, il est nécessaire de passer à une version de PHP plus récente qui n'est pas proposée dans le dépôt par défaut de Debian.
Une version à jour peut être obtenue ici :
https://codeberg.org/oerdnj/deb.sury.org

Utiliser le script d'installation fourni :
```bash
curl -sSL https://packages.sury.org/php/README.txt | sudo bash -x
sudo apt update
```
Ensuite, vérifier les paquets manquants :

```bash
sudo -u www-data php8.2 bin/console db:update
>Some mandatory system requirements are missing:
> - bcmath extension is missing
>Run the "php bin/console system:check_requirements" command for more details.
```

Installer les paquets manquants (adapter la liste au résultat précédent) :
```bash
sudo apt install -y php8.2-bcmath
```
Vérifier la mise à jour avec la commande :
```bash
sudo -u www-data php8.2 bin/console system:check_requirements
```
Source : https://help.glpi-project.org/faq/glpi/installation_update

S'assurer des bonnes permissions pour permettre la lecture du fichier downstream et par conséquent de la base de données :

```bash
sudo chown www-data:www-data /var/www/glpi/inc/downstream.php
```


Vérifier à nouveau si la mise à jour base de données fonctionne :
```bash
/var/www/glpi$ sudo -u www-data php8.2 bin/console db:update
>Some mandatory system requirements are missing:
> - Database engine version (10.5.29) is not supported. Minimum required version is MariaDB 10.6.
>Run the "php bin/console system:check_requirements" command for more details.
```

#### Mise à jour mariadb
Source : https://mariadb.com/docs/server/server-management/install-and-upgrade-mariadb/installing-mariadb/binary-packages/mariadb-package-repository-setup-and-usage#mariadb_repo_setup

```bash
curl -LsSO https://r.mariadb.com/downloads/mariadb_repo_setup
echo "${checksum} mariadb_repo_setup" | sha256sum -c -
chmod +x mariadb_repo_setup
sudo apt update
sudo apt install -y mariadb-server mariadb-client
sudo mariadb-upgrade
```

Conformément aux nouvelles directives de sécurité de GLPI 11, le point d'entrée web a été restreint au répertoire /public. Cette isolation empêche l'accès direct aux fichiers sensibles à la racine.

Configuration du VirtualHost Apache (/etc/apache2/sites-available/glpi.conf) :

```Apache
<VirtualHost *:80>
   DocumentRoot /var/www/glpi/public
   <Directory /var/www/glpi/public>
      AllowOverride All
      RewriteEngine On
      RewriteCond %{REQUEST_FILENAME} !-f
      RewriteRule ^(.*)$ index.php [QSA,L]
   </Directory>
</VirtualHost>
```
Activer le module "rewrite" :

```bash
sudo a2enmod rewrite
sudo a2ensite glpi.conf
sudo a2dissite 000-default.conf
sudo systemctl restart apache2
```

Rétablir la connectivité SQL.
Rétablir le fichier /etc/glpi/config_db.php pour lier l'application à la base MariaDB avec l'utilisateur glpi_admin pour restaurer l'accès à GLPI après la mise à niveau.

Relancer une migration :
```bash
cd /var/www/glpi
sudo -u www-data php8.2 bin/console db:update
```

Plus aucune erreur ne devrait s'afficher et la mise à jour devrait être réussie.




### Pour aller plus loin

Il existe une version Docker de GLPI qui pourrait s'avérer ou non pertinente. Le docker-compose.yaml ressemblerait à :

![Image](assets/20260118182627.png)

L'avantage majeur d'un déploiement Docker, notamment via docker-compose serait la facilité de mise à jour par rapport à la méthode manuelle décrite ci-dessus.
Avec un docker-compose qui va chercher le tag #latest, il suffit de lancer la commande docker compose pull et relancer le container. 

Source :
https://help.glpi-project.org/tutorials/fr/procedures/running_glpi_on_docker
Il suffirait alors de créer le fichier .env, adapter les bind mounts et ajouter un service pour un reverse proxy (nginx par exemple) ou ajouter un tunnel Cloudflared pour bénéficier des protections de Cloudflared.
