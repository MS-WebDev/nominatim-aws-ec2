# Nominatim auf Amazon (AWS) EC2 installieren

## AWS - EC2 Instanz erstellen

Diese Instanz dient auschließlich zur Einrichtung des Nominatim-Servers als Default-Image, welches später im Schnellverfahren aufgesetzt werden kann.

1. Neue Amazon EC2 Instanz erstellen
2. Unter Community AMIs ein Debian Jessie Image auswählen => ami-30e01d5f ([weitere Images](https://wiki.debian.org/Cloud/AmazonEC2Image/Jessie))
2. Als Instanztyp wähle t2.medium (kleinsmögliche Instanz für eine sichere Installation)
3. Alle weiteren Grundeinstellungen bestätigen, bis auf Configure Security Group
  * hier die Regel HTTP hinzufügen
4. wieder alles bestätigen und zum Abschluss noch die Schlüsseldatei für den SSH-Zugriff erzeugen

## Nominatim Quick-Install

* via SSH einloggen (Schlüsseldatei nicht vergessen)
* Setup-Script herunterladen und (WICHTIG: nur auf einer neuen Instanz s.o.) ausführen
```shell
cd /home/admin
wget https://raw.githubusercontent.com/MS-WebDev/nominatim-aws-ec2/master/setup-default-image.sh
chmod +x ./setup-default-image.sh
./setup-default-image.sh
```

## Nominatim Step-by-Step installieren

### Default-Image auf t2.medium einrichten

  * via SSH einloggen (Schlüsseldatei nicht vergessen)
  * Debian auf den neusten Stand bringen
```shell
sudo apt-get clean && sudo apt-get update && sudo apt-get dist-upgrade
```

  * Alle Abhängigkeiten für [Nominatim installieren](http://wiki.openstreetmap.org/wiki/Nominatim/Installation#Ubuntu.2FDebian)
```shell
sudo apt-get install build-essential libxml2-dev libpq-dev libbz2-dev libtool automake libproj-dev libboost-dev libboost-system-dev libboost-filesystem-dev libboost-thread-dev libexpat-dev gcc proj-bin libgeos-c1 libgeos++-dev libexpat-dev php5 php-pear php5-pgsql php5-json php-db libapache2-mod-php5 postgresql postgis postgresql-contrib postgresql-9.4-postgis-2.1 postgresql-server-dev-9.4 wget
```

  * Osmosis installieren
```shell
sudo apt-get install osmosis
```

  * Nominatim installieren
```shell
# Herunterladen und entpacken
wget http://www.nominatim.org/release/Nominatim-2.5.1.tar.bz2
tar xvf Nominatim-2.5.1.tar.bz2

# Kompilieren
cd Nominatim-2.5.1
./configure
make

# Zugriffsrechte ändern => module
chmod +x ~/Nominatim-2.5.1
chmod +x ~/Nominatim-2.5.1/module
```

  * PostgreSql Setup
```shell
# Passwort setzen
sudo -u postgres psql -c "ALTER USER postgres PASSWORD '00000';"
# user www-data zu Psql hinzufügen
createuser -SDR www-data
```

  * Webseite einrichten
```shell
sudo mkdir -m 755 /var/www/html/nominatim
sudo chown admin /var/www/html/nominatim

# Im Ordner: Nominatim-2.5.1
./utils/setup.php --create-website /var/www/html/nominatim
``` 

  * Nominatim Einstellungen setzen (./settings/local.php)
```shell
nano ./settings/local.php
```
```php
<?php
@define('CONST_Database_DSN', 'pgsql://postgres:00000@localhost:5432/nom_sl');
@define('CONST_Postgresql_Version', '9.4');
@define('CONST_Website_BaseURL', 'http://public-ip/nominatim/');

# Update-Einstellungen abgestimmt auf geofabrik.de
@define('CONST_Replication_Url', 'http://download.geofabrik.de/europe/germany/saarland-updates');
@define('CONST_Replication_MaxInterval', '40000');
@define('CONST_Replication_Update_Interval', '86400');
@define('CONST_Replication_Recheck_Interval', '900');
```

### Testlauf (ohne Update-Funktion)
```shell
# Extrakt herunterladen
wget -N http://download.geofabrik.de/europe/germany/saarland-latest.osm.pbf -P /home/admin/Nominatim-2.5.1/data

# Import starten
./utils/setup.php --osm-file ./data/saarland-latest.osm.pbf --all
``` 
  
  * Nachdem das Test-Extract ohne Fehler importiert wurde, kann die Suche über http://public-ip/nominatim getestet werden
  
### Nominatim vor der Image-Erstellung säubern
```shell
# Test-Datenbank entfernen
sudo -u postgres psql -c "DROP DATABASE nom_sl;"

# Abhängige Dateien entfernen
rm /home/admin/Nominatim-2.5.1/settings/state.txt 
rm /home/admin/Nominatim-2.5.1/settings/configuration.txt 
rm /home/admin/Nominatim-2.5.1/settings/download.lock
``` 
### Default-Image über die AWS Konsole erstellen

  * In der AWS Konsole diese Instanz auswählen
  * über Actions kann jetzt ein Image erstellt werden
  * Image name: nom-2.5.1-default (frei wählbar)
  * WICHTIG: In der Spalte "Delete on Termination" muss der Haken entfernt werden, sonst wird beim Löschen der Instanz dieses Image ebenfalls gelöscht
  * Fertig: dieses Image kann jetzt auf ein beliebigen Instanztyp angewendet werden. Für große Imports bieten sich die [R3-Typen](https://aws.amazon.com/de/ec2/instance-types/#r3) an
  
## Hilfreiche Befehle
  * Datenbank entfernen
```shell
sudo -u postgres psql -c "DROP DATABASE db_name;"
```
  
  * Nominatim-Einstellungen ändern (local.php)
```shell
nano /home/admin/Nominatim-2.5.1/settings/local.php
```

  * PostgreSql Befehle
```shell
# Default Postgresql.conf laden (wichtige Variablen oben) 
sudo wget --output-document=/etc/postgresql/9.4/main/postgresql.conf https://raw.githubusercontent.com/MS-WebDev/nominatim-aws-ec2/master/postgresql.conf
# PostgreSql Konfiguration bearbeiten
sudo nano /etc/postgresql/9.4/main/postgresql.conf
# Restart PSQL-Server
sudo service postgresql restart
```


