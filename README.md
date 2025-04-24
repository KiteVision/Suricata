# Installation de SURICATA sur une machine Linux Debian 12
![image](https://github.com/user-attachments/assets/dcf2fa33-8449-4ede-96a6-2d6ae8204954)

Un simple server Debian permet de tester SURICATA, avec une visualisation des logs à l’aide d’EVEBOX.

## Machine utilisée pour l’installation

Une image de machine virtuelle Debian 12 au format ova est disponible à l’url ci-dessous :  
https://www.kitevision.fr/VMs/Debian-Server.ova

La vérification de l’intégrité de l’image chargée peut être réalisée en calculant son hash SHA1 et en vérifiant qu’il correspond bien à celui calculé dans ce fichier :  
https://www.kitevision.fr/VMs/Debian-Server.sha1

Cette machine virtuelle comprend 2 interfaces réseau et on veillera à ce que l’une des deux interfaces, disons ens33, soit en mode « host only » et que l’autre, disons ens34, soit en mode « bridge » et connectée au WAN.

Nous configurerons la capture sur l’interface connectée au WAN, ens34 dans l’exemple ci-dessus donc.
![image](https://github.com/user-attachments/assets/fd6f42c8-e6d6-41e2-a942-ca03f31e54fd)

Une fois démarrée, on peut changer le nom d’hote de cette machine, en la nommant IDS par exemple ;

hostnamectl set-hostname IDS

et en ajoutant également ce nom d'hote dans le fichier /etc/hosts, sur l'une des interfaces de la machine.

## Installation de Suricata

L’application étant disponible dans les packages standards de Debian, nous l’installons simplement avec :

sudo apt install suricata

Avant de la démarrer, procédons à une configuration minimale de l’application :

Précisons quelle interface réseau sera supervisée avec Suricata en éditant le fichier /etc/suricata/suricata.yaml et en modifiant l’interface précisée dans la section af-packet (en remplaçant 'ens34' par l'interface à monitorer) :

nano /etc/suricata/suricata.yaml
![image](https://github.com/user-attachments/assets/408d2360-e5a0-4d3b-b188-a2e12fd26e34)

Il est possible déclarer d’autres interfaces de captures, avec ce meme format et en ajoutant un cluster-id distinct pour chaque interface.

Chargeons les règles de détection proposées par défaut, grâce à l’utilitaire suricata-update, installé avec le package suricata :  
sudo suricata-update
![image](https://github.com/user-attachments/assets/965b4a50-2dbe-4f8e-8ac5-168450d9e4e3)

ATTENTION : 
Il est possible que suricata-update crée le fichier de règles "suricata.rules" dans le répertoire /var/lib/suricata/rules,
si c'est le cas il faut le copier dans /etc/suricat/rules :

cp /var/lib/suricata/rules/suricata.rules /etc/suricata/rules/.

Et nous pouvons alors démarrer l’application, configurée en service avec systemd :

sudo systemctl start suricata.service

Puis vérifier que le service est bien actif, sans erreur :

sudo systemctl status suricata.service
![image](https://github.com/user-attachments/assets/e9f1ad57-abc3-4870-87eb-dcbb1df1fd9a)


Si tout est OK, il ne reste que plus qu’à activer son démarrage automatique :

sudo systemctl enable suricata.service

## Examen des logs Suricata

Les différents logs produits par l’application se trouvent sous /var/log/suricata.

On s’intéressera en particulier au fichier /var/log/suricata/eve.json qui compile le résultat des analyses de trafic.

Un test simple pour vérifier le bon fonctionnement de SURICATA peut être réalisé ainsi :

curl http://testmynids.org/uid/index.html

Cette transaction http déclenchera une alerte.

## Installation de EveBox

EveBox simplifie grandement la consultation des logs, installons l’application ainsi :

```
sudo apt-get install curl
curl -fsSL https://evebox.org/files/GPG-KEY-evebox -o /etc/apt/keyrings/evebox.asc
echo "deb [signed-by=/etc/apt/keyrings/evebox.asc] https://evebox.org/files/debian stable main" | sudo tee /etc/apt/sources.list.d/evebox.list
sudo apt-get update
sudo apt-get install evebox
```

Et lançons le serveur EveBox sur le port 8081 en http :

```
evebox server --no-auth --no-tls --sqlite /var/log/suricata/eve.json --host 0.0.0.0 --port 8081 > /var/log/evebox-server.log 2>&1 &
```

Nous pouvons alors accéder à l’IHM d’EveBox et naviguer dans les logs issus du fichier eve.json :

![image](https://github.com/user-attachments/assets/613fb23f-1a0f-4bd4-ac99-089a8e130baa)


Nous disposons ainsi d’un IDS, que nous allons pouvoir tester en simulant attaques ou scans.
