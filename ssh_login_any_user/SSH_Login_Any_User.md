# QNAP QTS 4.2 - SSH Login für beliebige Nutzer


## Problemstellung

Die Standardeinstellungen des QTS erlauben nicht die SSH Anmeldung für beliebige Nutzer. Vor dem Hintergrund des Aufbaus eines 
Backupservers, dass das _Rsync_ oder _Rsnapshoot_ Protokoll nutzt wäre ein SSH Zugang höchst wünschenswert.


## Vorraussetzung

* QNAP NAS
* Putty

## QTS (Server)
### Erweiterte Ordnerzugriffsrechte

Standardmäßig ist beim dem QTS Betriebssystem die vereinfachte Ordnerfreigabe aktiviert. Die führt während des Betriebes immer wieder zu dem 
überschreiben der mittles _chmod_ gesetzten Berechtigungen. Dies ist dahingehend ärgerlich, da der SSH Deamon bestimmte Ordnerberechtigungen
benötigt.

_QTS -> Systemsteuerung -> Privelegieneinstellungen -> Freigabe-Ordner -> Erweiterte Ordnerzugrifssrechte aktivieren_ 

<br/>
<center><img src=" /inc/advancedDirectoryPermissions.png" height="100%" width="100%" alt="Screenshot" title="Aktivierung erweiterter Ordnerzugriffsrechte" /></center>
<br/>



### Erzeugung RSA Schlüssel

Zunächst ist ein RSA Schlüsselpaar zu erzeugen. Nach _[SSH How To Set Up Authorized Keys][]_ folgt für den Befehl:
```sh
ssh-keygen -t rsa -C "My Amazing SSH Server"
``` 

Der Parameter _-C_ erlaubt die Vergabe eines Kommentars der später in der _authorized\_keys_ auftaucht. Nach dem Aufruf des Kommandos erfolgt die Ausgabe:
```sh
# Generating public/private rsa key pair. 
# Enter file in which to save the key (/Users/UserName/.ssh/id_rsa):
``` 

Zusätzlich kann der Schlüssel noch mit einem Passwort gesichert werden. Bei einfachem Bestätigen erfolgt keine Passwortvergabe:
```sh
# Enter passphrase (empty for no passphrase):
# Enter same passphrase again:
``` 

Nun sollten folgende beide Dateien vorhanden sein:
* _id\_rsa_ 		(privater Schlüssel)
* _id\_rsa.pub_		(öffentlicher Schlüssel)



### Schlüssel dem Nutzer (_myUser_) hinzufügen

Als nächstes in das _Home_ Verzeichnes des Nutzer mit zukünftigen SSH Zugang navigieren (Bei QTS in der Regel):
```sh
cd /share/homes/<myUser>
``` 

Falls noch nicht vorhanden das Verzeichnis _.ssh_ anlegen:
```sh
mkdir .ssh
``` 

Dort der _authorized\_keys_ den Inhalt des Schlüssels _id\_rsa.pub_ hinzufügen:
```sh
cat id_rsa.pub >> authorized_keys
``` 

Nun sind die Berechtigungen wie folgt anzupassen:
```sh
chmod 0711 /share/homes/<myUser>
chmod 0700 /share/homes/<myUser>/.ssh
chmod 0600 /share/homes/<myUser>/.ssh/authorized_keys
``` 

Die Nutzung des _Admin_ SSH Zuganges bedingt das sämtliche angelegte Dateien nun dem Benutzer _Admin_ gehören. Damit der SSH login funktionieren kann,
müssen die Besitzrechte und die Gruppenzugehörigkeit bei dem über SSH zu authentifizierenden Nutzer liegen:
```sh
chown myUser /share/homes/<myUser>/.ssh
cd /share/homes/<myUser>/.ssh
chown myUser ./
chown myUser authorized_keys
``` 

Ändern der Gruppenzugehörigkeit:
```sh
chgrp myUserGrp /share/homes/<myUser>/.ssh
cd /share/homes/<myUser>/.ssh
chgrp myUserGrp ./
chgrp myUserGrp authorized_keys
``` 

Die Ausgabe von _ls -la_ sollte wie folgt aussehen:
```sh
drwx------    2 <myUser> <myUserGrp>      4096 Dec 23 21:18 ./
drwx------    4 <myUser> <myUserGrp>      4096 Dec 23 21:32 ../
-rw-------    1 <myUser> <myUserGrp>       417 Dec 23 21:18 authorized_keys
``` 


### Konfiguration SSH Deamon

Zur Einrichtung des SSH Deamons empfiehlt sich die Einrichtung eines zweiten Servers mit separaten Port und eigener Konfiguration Die Vorteile dafür liegen:
* Port Weiterleitung kann nur für den Deamon der unpreveligierte Benutzer hostet erfolgen
* Erzwingen einer RSA Authentifizierung durch Unterbindung der Authentifizierung mittels Passwort

Anlegen der zusätzlichen SSH Konfiguration
```sh
mkdir -p /opt/etc/ssh
cd /opt/etc/ssh
cp /etc/ssh/sshd_config ./
mv sshd_config sshd_port40_config
``` 

Öffnen der Datei zur Editierung mittels _nano sshd\_config_ und folgende Anpassungen durchführen
* Ändern des Standardportes, auf z.B. Port 40 ([Standardisierte Ports](https://de.wikipedia.org/wiki/Liste_der_standardisierten_Ports "Liste der standardisierten Ports"))
* Unterbinden der Passwortauthentifizierung
* Verbieten des _Root_ login
* Angabe des Pfades zu den erlaubten öffentlichen RSA Schlüsseln

```sh
PasswordAuthentication no
Port 40
PermitRootLogin yes
StrictModes no
MaxAuthTries 3
AuthorizedKeysFile      .ssh/authorized_keys
AllowUsers <myUser>
``` 

Den SSH Server im Debugmodus starten:
```sh
/usr/sbin/sshd -ddd -f /opt/etc/ssh/sshd_port40_config
``` 

In der Konsole sollten folgende Meldungen des SSH Servers zuletzt stehen:
```sh
debug1: Bind to port 40 on 0.0.0.0.
Server listening on 0.0.0.0 port 40.
```


### Einrichten Init.d Script

Um den automatischen Start des SSH Deamons zu gewährleisten ist ein _init.d_ Skript (z.B. _S40ssh\_\<myPort\>_) unter _/opt/etc/init.d_ anzulegen. 
Das [_init.d_ Initialisierungsskript]( /inc/S40ssh_port40 "Skript zum Start des SSH Deamons") bewältigt dabei folgende Aufgaben:
* Start des SSH Deamons
* Anlegen des _PID_-Files
* Bereitstellung _start_/_stop_ Befehl

Zusätzlich ist die [automatische Skriptausführung](/linux/qnap_qts/automatic_script_run_at_startup/Automatic_Script_Run_At_Startup.md "Optware Init.d Autorun") 
des Ordners _init.d_ nachdem Systemhochlauf einzurichten.


## Putty (Client)


## Beobachtete Probleme

* Nachdem SSH Login bekommt das _Home_ Verzeichnis des angemeldeten Benutzer, die Berechtigungen 777. Der SSH Deamon prüft vor dem Login die Berechtigungen, und erwartet für das _Home_
Verzeichnis 700. Da diese nun aber auf 777 stehen verweigert der Deamon jeden weiteren Zugang. Die Lösung erfolgte aktuell über das Abschalten dieser Prüfung. In der Konfiguration
des SSH Deamons sieht das wie folgt aus:

```sh
StrictModes no
``` 


## Referenzen

<!--- Internetlinks -->
[SSH How To Set Up Authorized Keys]:												https://wiki.qnap.com/wiki/SSH:_How_To_Set_Up_Authorized_Keys   														"Anleitung zur Erzeugung eines RSA-Schlüssels und Einstellung des SSH Zuganges"
[Advanced Folder Permissions on QNAP NAS]:											https://www.qnap.com/en/tutorial/con_show.php?op=showone&cid=6															"Deaktivierung der vereinfachten Dateiberechtigungen unter Linux"
[SSH Server refused our key]:														https://forum.qnap.com/viewtopic.php?t=33942																			"Server lehnt RSA Authentifizierung ab"
[SSH broken after homedir permissions and hostname change on EC2-hosted Ubuntu]:	http://serverfault.com/questions/452034/ssh-broken-after-homedir-permissions-and-hostname-change-on-ec2-hosted-ubuntu	"Konfigurationsanpassung des SSH deamons"