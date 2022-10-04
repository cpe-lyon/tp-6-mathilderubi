# Compte-rendu TP 6
*Mathilde Rubi*

## Exercice 1. Adressage IP

L'adresse du sous-réseau 1 est 172.16.0.0/26, son adresse de broadcast est 172.16.0.63/26, l'adresse de la première machine est 172.16.0.1/26, l'adresse de la dernière machine est 172.16.0.39/26.

L'adresse du sous-réseau 2 est 172.16.0.64/26, son adresse de broadcast est 172.16.0.127/26, l'adresse de la première machine est 172.16.0.65/26, l'adresse de la dernière machine est 172.16.0.98/26.

L'adresse du sous-réseau 3 est 172.16.0.128/26, son adresse de broadcast est 172.16.0.191/26, l'adresse de la première machine est 172.16.0.129, l'adresse de la dernière machine est 172.16.0.181.

L'adresse du sous-réseau 4 est 172.16.0.192/26, son adresse de broadcast est 172.16.0.255/26, l'adresse de la première machine est 172.16.0.193/26, l'adresse de la dernière machine est 172.16.0.228/26.

L'adresse du sous-réseau 5 est 172.16.1.0/26, son adresse de broadcast est 172.16.1.63/26, l'adresse de la première machine est 172.16.1.1, l'adresse de la dernière machine est 172.16.1.35.

L'adresse du sous-réseau 6 est 172.16.1.64/26, son adresse de broadcast est 172.16.1.127/26, l'adresse de la première machine est 172.16.1.65, l'adresse de la dernière machine est 172.16.1.102.

L'adresse du sous-réseau 7 est 172.16.1.128/26, son adresse de broadcast est 172.16.1.190/26, l'adresse de la première machine est 172.16.1.129, l'adresse de la dernière machine est 172.16.1.154.

On a la possibilité de configurer un réseau en plus, ça serait un sous-réseau 8 en 172.16.1.192/26, son adresse de broadcast est 172.16.1.255/26.

## Exercice 2. Préparation de l'environnement

1. Pour configurer les cartes réseaux du serveur, on ouvre les outils de configuration de Virtual Box, on va dans réseau, on configure une première carte réseau en accès NAT pour avoir un accès internet et on en ouvre une deuxième qu'on configure en accès en réseau privé hôte pour qu'elle puisse communiquer avec la machine cliente. On configure ensuite la carte réseau du client en accès en réseau privé hôte.

2. L'interface appelée lo est l'interface de loopback, habituellement l'adresse de localhost.

3. Pour désinstaller le paquet cloud-init on effectue la commande `sudo apt purge cloud-init`.

4. Pour renommer la machine en "server" on exécute la commande `sudo hostname set-hostname server.tpadmin.local`. Puis on va modifier le fichier /etc/hosts :
    `nano /etc/hosts`
On commente la deuxième ligne et on remplace localhost par server server.tpadmin.local, on enregistre et on quitte.
On reboot et on vérifie en exécutant la commande `hostname`, la machine nous indique bien server.tpadmin.local.
    
## Exercice 3. Installation du serveur DHCP

1. Pour installer le paquet on exécute `sudo apt install isc-dhcp-server`. La commande `systemctl status isc-dhcp-server` indique bien que le serveur n'a pas réussi à démarrer.

2. Pour attribuer des adresses IP permanentes aux cartes réseau, on va modifier le netplan avec les commandes suivantes. D'abord, il nous faut connaitre le nom de la carte réseau du réseau interne de la machine serveur, on fait donc un ip a et on voit que l'interface se nomme enp0s8. On va donc modifier le netplan pour attribuer à cette interface une adresse IP : `sudo nano /etc/netplan/(ici on effectue une tabulation pour que le système trouve lui même le fichier)`
On se met sous la ligne du dhcp, aligné avec la première carte réseau qui est l'interface en NAT, qui a donc accès à internet, on utilise que des retours à la ligne et des espaces. On inscrit le nom de la carte réseau et son adresse, ainsi :

`enp0s8:
   addresses:
       - 192.168.100.1/24
`

On sauvegarde et on quitte, on fait ensuite un `netplan try`puis un `sudo netplan apply`.
On vérifie la configuration en faisant un ip a.
En faisant la même chose avec le client, en supprimant la ligne dhcp et en indiquant une adresse sur sa seule interface réseau : 

`
enp0s3:
   addresses:
       - 192.168.100.2/24
`

On pourra ensuite essayer de ping l'une à partir de l'autre pour vérifier qu'elles communiquent.

3. Pour configurer le serveur DHP on commence par copier le fichier dhcpd.conf en dhcpd.conf.bak : `sudo cp   /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.bak`.
   Puis on édite le fichier dhcpd.conf avec les informations :
   `default-lease-time 120;
    max-lease-time 600;
    authoritative; #DHCP officiel pour notre réseau
    option broadcast-address 192.168.100.255; #informe les clients de l'adresse de broadcast
    option domain-name "tpadmin.local"; #tous les hôtes qui se connectent au réseau auront ce nom de domaine
    subnet 192.168.100.0 netmask 255.255.255.0 { #configuration du sous-réseau 192.168.100.0
    range 192.168.100.100 192.168.100.240; #pool d'adresses IP attribuables
    option routers 192.168.100.1; #le serveur sert de passerelle par défaut
    option domain-name-servers 192.168.100.1; #le serveur sert aussi de serveur DNS
  }`
  
  Les deux premières lignes sont les durées minimales et maximales de vie des adresses IP. Lorsque la durée maximale est dépassée, le serveur DHCP distribuera une nouvelle adresse IP.
  
4. On édite ensuite le fichier `/etc/default/isc-dhcp-server` pour spécifier l'interface sur laquelle le serveur doit écouter. On ajoute à INTERFACESv4="enp0s8" la carte réseau enp0s8 correspondant à l'interface réseau du réseau interne. 
5. On exécute la commande dhcpd -t pour valider les configurations puis on redémarre le serveur DHCP
(avec la commande systemctl restart isc-dhcp-server). On remet le client en DHCP, on remplace la ligne address par `dhcp4 : true`. On exécute la commande `sudo netplan apply` pour appliquer la configuration. On effectue un `ip a` pour vérifier que le client ait bien une adresse IP. On essaye de ping le serveur pour vérifier que ça fonctionne.

6. De la même manière que pour la machine serveur, pour renommer la machine on exécute la commande `sudo hostname set-hostname client.tpadmin.local`
On modifie ensuite le fichier /etc/hosts :
`sudo nano /etc/hosts`
On commente la deuxième ligne et on remplace localhost par client client.tpadmin.local, on enregistre et on quitte.
On reboot et on vérifie en exécutant la commande `hostname`, la machine nous indique bien client.tpadmin.local.

7. Pour désactiver la carte réseau du client, on fait `sudo ip link set enp0s3 down`. On va écouter sur le serveur avec la commande `tail -f /var/log/syslog` puis on réactive la carte réseau du client avec la commande `sudo ip link set enp0s3 up`. On constate que des lignes sont apparues sur le serveur. Le DHCPDISCOVER sert à détecter les serveurs DHCP disponibles. Le DHCPOFFER répond à un paquet DHCPDISCOVER, DHCPREQUEST est une requête diverse du client pour par exemple prolonger son bail, DHCPACK est la réponse du serveur qui contient des paramètres et l'adresse IP du client. On fait un `ip a` pour regarder si le client a une adresse ip comprise dans la plage indiquée. C'est le cas, le client a l'adresse IP 192.168.100.100.

8. Le fichier /var/lib/dhcp/dhcpd.leases contient les logs du système. La commande `dhcp-lease-list`affiche le fichier dhcp.leases.

9. En faisant un `ping 192.168.100.100` sur la machine serveur vers la machine client et un `ping 192.168.100.1` de la machine client vers la machine serveur, on constate que les deux machines peuvent bien communiquer.

10. Pour que l'interface réseau client reçoive l’IP statique 192.168.100.20, on modifie le fichier /etc/dhcp/dhcpd.conf `sudo nano /etc/dhcp/dhcpd.conf`; on ajoute les lignes à la fin du fichier : 

`deny unknown-clients; #empêche l'attribution d'une adresse IP à une
#station dont l'adresse MAC est inconnue du serveur
host client {
hardware ethernet XX:XX:XX:XX:XX:XX; #remplacer par l'adresse MAC
fixed-address 192.168.100.20;
}
`
On exécute la commande `sudo dhcpd -t` pour appliquer les changements puis on redémarre ensuite le serveur DHCP : `systemctl restart isc-dhcp-server`
On vérifie ensuite si le serveur dhcp est actif : `systemctl status isc-dhcp-server`. 
Puis sur la machine cliente on désactive et réactive la carte réseau pour qu'elle envoie une requête au serveur DHCP et qu'elle obtienne une nouvelle adressse IP : `sudo ip link set enp0s3 down` puis `sudo ip link set enp0s3 up`. On vérifie que le client ait bien l'adresse 192.168.100.20 en faisant la commande `ip a`.

## Exercice 4
