# Instalace a konfigurace Shibboleth a Jetty pro UTIA AV ČR

## Server
Minimální instalace Rocky 9, 2x CPU 4G RAM
Po instalci standardní kroky pro atyp instalce, uodate, zákaz selinuxu, instalace potřebných věcí a javy
```
yum -y update
yum -y install java-17-openjdk.x86_64
yum -y install wget
yum -y install openldap-clients
yum -y install firewall-config
yum -y install xauth
alternatives --config java
Type java
eval $(echo "export JAVA_HOME=/usr" | tee -a /root/.bashrc)
```
Napsat disabled do:
```
vi /etc/sysconfig/selinux
```
a reboot.

## Mariadb
Instalace:
```
yum -y install mariadb-server mariadb-java-client mariadb
```
Konfigurace:
```
systemctl enable mariadb
systemctl start mariadb
mysql_secure_installation
```
Helso na mariadb z předchozí instalace a nebo jiné dle chuti kde kterého soudruha.
Dále je nutné založit databázi shibboleth, uživatele shibboleth, přenést data z předchozí instalace a založit tabulku `StorageRecords`.
Takže na starém serveru se udělá:
```
mysqldump -u root -p shibboleth > ~/persistentIDs.sql
```
Na novém:
```
mysql -u root -p
SET NAMES 'utf8';
SET CHARACTER SET utf8;
CHARSET utf8;
CREATE DATABASE IF NOT EXISTS shibboleth CHARACTER SET=utf8;
USE shibboleth;
CREATE TABLE IF NOT EXISTS `StorageRecords` (
  context varchar(255) COLLATE utf8mb4_bin NOT NULL,
  id varchar(255) COLLATE utf8mb4_bin NOT NULL,
  expires bigint DEFAULT NULL,
  value text NOT NULL,
  version bigint NOT NULL,
  PRIMARY KEY (context, id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
DESCRIBE StorageRecords;
GRANT ALL PRIVILEGES ON shibboleth.*
      TO 'shibboleth'@'localhost'
      IDENTIFIED BY 'SILNE_HESLO';
FLUSH PRIVILEGES;
```
a pak nahrátí starých dat
```
mysql -u root -p shibboleth < ~/persistentIDs.sql
```

Hotov
