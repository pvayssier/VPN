# VPN

# Présentation du projet

Ce projet a pour but de créer un VPN (Virtual Private Network) à l'aide de WireGuard sur une machine Rocky8 avec une interface.

# 1. Installation des paquets

Commençons par installer les paquets dont on aura besoin lors de la réalisations du TP.

```
sudo dnf install elrepo-release epel-release
sudo dnf install kmod-wireguard wireguard-tools
sudo dnf install firewalld
```
---
Lançons le service de firewalld 
```
sudo systemctl enable firewalld
sudo systemctl start firewalld
```

# 2. Générer les clés

Nous devons générer les clés afin de chiffrer de bout en bout les données qui passerons par notre tunnel.
```
wg genkey | sudo tee /etc/wireguard/private.key
sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
```
Nous devons aussi changer les permissions d'accès au fichier contenant notre clés privées

```
sudo chmod go= /etc/wireguard/private.key
```

# 3. Configurations de WireGuard

Nous allons ajouter le fichier de configuration /etc/wireguard/wg0.conf

```
sudo vi /etc/wireguard/wg0.conf
```
En y ajoutant le les informations suivante

```
[Interface]
PrivateKey = Votre_clé_privée
Address = Ip
ListenPort = 51820
SaveConfig = true
```
- Prenons bien le temps de modifier `Votre_clé_privée` par la clé privée que nous avons généré au préalable qui est dans /etc/wireguard/private.key et modifier `Ip` par l'ip que vous voulez donner à votre machine dans le réseau de votre vpn. Moi, je choisis de mettre `10.55.0.1/24`. N'oubliez pas de préciser le masque.
  
- Ensuite, nous allons modifier le fichier `/etc/sysctl.conf`, en ajoutant à la fin du fichier `net.ipv4.ip_forward=1`. Pour voir si la modification a été prise en compte, faisons `sudo sysctl -p` et cela doit afficher la ligne que l'on vient d'ajouter.

# 4. Modification du firewall de votre serveur

Le serveur VPN va écouter sur le port *51820* donc il faut ouvrir ce port
```
sudo firewall-cmd --zone=public --add-port=51820/udp --permanent
```
Ensuite nous allons créer une nouvelle interface pour WireGuard afin que tout le flux du VPN passe par celui ci.
```
sudo firewall-cmd --zone=internal --add-interface=wg0 --permanent
```
Nous devons aussi, comme nous allons utiliser notre serveur comme un routeur, ajouter quelques règles au firewall. Modifier la ligne en ajoutant le réseau dans lequel vous avez mis votre serveur avec le masque à la place de `IP`
```
sudo firewall-cmd --zone=public --add-rich-rule='rule family=ipv4 source address=IP masquerade' --permanent
firewall-cmd --add-masquerade
firewall-cmd --add-masquerade --permanent
```
Enfin, nous relançons le firewall pour que toutes modifications soient prises en comptes.
```
sudo firewall-cmd --reload
```
Pour voir si tout est bon faisons:
```
sudo firewall-cmd --list-all
```
Et vous devriez avoir comme moi. Bien sur vous devez avoir l'IP de votre réseau à la place de `IP_De_Votre_Réseau`.
```
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0 eth1
  sources:
  services: cockpit dhcpv6-client ssh
  ports: 51820/udp
  protocols:
  masquerade: yes
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
    rule family="ipv4" source address="IP_De_Votre_Réseau" masquerade
```
Testons aussi si l'interface est bien créée:
```
sudo firewall-cmd --zone=internal --list-interfaces
```
Ça doit vous retourner `gw0`

# 5. Lançons le service

Maintenant que tout est bien paramétré, nous pouvons lancer le service

```
sudo systemctl enable wg-quick@wg0.service
sudo systemctl start wg-quick@wg0.service
```

Voyons voir si tout est bon:
```
systemctl status wg-quick@wg0.service
```
```
● wg-quick@wg0.service - WireGuard via wg-quick(8) for wg0
   Loaded: loaded (/usr/lib/systemd/system/wg-quick@.service; enabled; vendor preset: disabled)
   Active: active (exited) since Fri 2021-09-17 19:58:14 UTC; 6 days ago
     Docs: man:wg-quick(8)
           man:wg(8)
           https://www.wireguard.com/
           https://www.wireguard.com/quickstart/
           https://git.zx2c4.com/wireguard-tools/about/src/man/wg-quick.8
           https://git.zx2c4.com/wireguard-tools/about/src/man/wg.8
 Main PID: 22924 (code=exited, status=0/SUCCESS)
    Tasks: 0 (limit: 11188)
   Memory: 0B
   CGroup: /system.slice/system-wg\x2dquick.slice/wg-quick@wg0.service

Sep 17 19:58:14 wg0 systemd[1]: Starting WireGuard via wg-quick(8) for wg0...
Sep 17 19:58:14 wg0 wg-quick[22924]: [#] ip link add wg0 type wireguard
Sep 17 19:58:14 wg0 wg-quick[22924]: [#] wg setconf wg0 /dev/fd/63
Sep 17 19:58:14 wg0 wg-quick[22924]: [#] ip -4 address add 10.8.0.1/24 dev wg0
Sep 17 19:58:14 wg0 wg-quick[22924]: [#] ip -6 address add fd0d:86fa:c3bc::1/64 dev wg0
Sep 17 19:58:14 wg0 wg-quick[22924]: [#] ip link set mtu 1420 up dev wg0
Sep 17 19:58:14 wg0 systemd[1]: Started WireGuard via wg-quick(8) for wg0.
```

OK tout es bon 🎉


# 6. Connectons un client

Nous pouvons télécharger l'application qui permet de simplifier toute la configuration du client. Il faudra simplement ajouter un tunnel vide et y ajouter quelques informations. Modifier `Clé_Privée_Client` par la clé de votre client et `IP_Client` par l'ip que vous voulez donner à votre client et mettez `/32`. Modifier `Clé_Public_Serveur` par la clé public de votre serveur vpn et `IP_Serveur` par l'ip que vous utilisez pour vous connecter en ssh.

```
[Interface]
PrivateKey = Clé_Privée_Client
Address = IP_Client/32
DNS = 1.1.1.1, 8.8.8.8

[Peer]
PublicKey = Clé_Public_Serveur
AllowedIPs = 0.0.0.0/0
Endpoint = IP_Serveur:51820
```

Il faut juste ajouter le client au serveur avec la commande en modifiant la clé public et l'ip dans le réseau du client:
```
sudo wg set wg0 peer Clé_Public_Client allowed-ips IP_Assigné_Client
```

# 7. Interface

Maintenant que nous savons comment faire, nous allons simplifier la partie client avec une interface web qui permet de gérer les clients simplement. Nous allons passer par wireguard-ui. 
- Pour commencer, vous devez cloner [wireguard-ui](https://github.com/ngoduykhanh/wireguard-ui)
- Modifier la ligne 29 du fichier `main.go` en remplaçant `"0.0.0.0:5000"` par `"127.0.0.1:5000"`
- Nous devons ensuite compiler le programme
    * Commencez par installer yarn (`sudo dnf install yarn go`)
    * Lancer la commande `sudo ./prepare_assets.sh`
    * Ensuite `go build -o wireguard-ui`
    * Enfin un `sudo chmod -R 700 wireguard-ui`
- Maintenant il faut mettre en place le reverse-proxy
    * Il faut installer nginx (`sudo dnf install nginx`)
    * Modifier le fichier `/etc/nginx/nginx.conf` avec [ce fichier](nginx.conf)
    * Créer deux dossiers, `/etc/nginx/sites-enabled` et `/etc/nginx/sites-available`
    * Créer et modifier le fichier `/etc/nginx/sites-available/vpn.conf` avec [ce fichier](vpn.conf)
    * Faites un lien entre deux fichiers, `sudo ln /etc/nginx/sites-available/vpn.conf /etc/nginx/sites-enabled/`
- Il faut se créer un certificat  
    * Allez dans le fichier `/etc/ssl/certs/`
    * Exécutez la commande 
```
openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout server.key -out server.crt
```
- N'oublions pas le firewall
    * `sudo firewall-cmd --add-port=443/tcp --permanent`
    * `sudo firewall-cmd --add-port=80/tcp --permanent`
---
Normalement tout est bon, il ne reste plus qu'à lancer le service nginx

```
sudo systemctl enable nginx
sudo systemctl start nginx
```

Maintenant il reste plus qu'à lancer le site.
```
cd wireguard-ui
sudo ./wireguard-ui &
```
**Bravo vous avez fait un vpn avec son interface graphique**
![vpn please](pics/meme_cat.jpg)