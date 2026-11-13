# Notes de conception

[English](design-notes.md) | **Français**

## Router-on-a-stick

Les cinq VLAN partagent un seul lien physique entre R1 (`Gi0/0`) et le switch (`Gi0/1`), transporté en trunk 802.1Q. R1 a une sous-interface par VLAN, marquée avec l'ID du VLAN, et chacune sert de passerelle par défaut pour son VLAN. C'est peu coûteux et suffisant pour un site de cette taille. L'inconvénient, c'est que tout le trafic entre VLAN passe par ce lien unique ; dans un réseau plus grand, l'étape suivante serait un switch de niveau 3.

## Les ACL

Ce sont des ACL étendues, et elles sont sans état : chaque paquet est jugé seul et le routeur ne garde pas la mémoire des connexions. La direction compte donc. `in` filtre les paquets qui arrivent sur une interface, `out` filtre ceux qui en sortent.

### ACL 100 : invités (`Gi0/0.50`, in)

```
access-list 100 deny ip 192.168.50.0 0.0.0.255 192.168.0.0 0.0.255.255
access-list 100 permit ip 192.168.50.0 0.0.0.255 any
```

Elle est placée au plus près des invités, donc le trafic vers n'importe quel réseau interne est bloqué avant d'être routé. Tout le reste passe, c'est-à-dire Internet dans ce lab. Un effet de bord : la passerelle des invités (192.168.50.1) est aussi dans 192.168.0.0/16, donc les invités ne peuvent probablement pas pinguer leur propre passerelle (je ne l'ai pas testé).

### ACL 110 : Finance (`Gi0/0.30`, out)

```
access-list 110 permit ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
access-list 110 deny ip 192.168.0.0 0.0.255.255 192.168.30.0 0.0.0.255
access-list 110 permit ip any any
```

Seul IT peut envoyer du trafic vers Finance. Toute autre source interne est refusée. Comme l'ACL est en sortie vers Finance, elle protège le trafic qui va vers Finance. Les hôtes Finance peuvent quand même initier des connexions vers d'autres VLAN. Pour la plupart, les réponses sont bloquées au retour, puisque seul IT passe, et c'est le comportement sans état qui apparaît. La dernière ligne laisse passer le trafic de retour d'Internet.

### ACL 120 : serveurs (`Gi0/0.40`, out)

```
access-list 120 permit tcp any 192.168.40.0 0.0.0.255 eq www
access-list 120 permit udp any 192.168.40.0 0.0.0.255 eq domain
access-list 120 deny ip any 192.168.40.0 0.0.0.255
access-list 120 permit ip any any
```

Seuls HTTP et DNS atteignent le VLAN des serveurs. Tout le reste, y compris le ping et le SSH, est bloqué. Deux limites : le HTTPS (443) n'est pas autorisé alors que le serveur l'a activé, et le personnel IT ne peut pas non plus joindre les serveurs pour les administrer. Un permit réservé à IT au-dessus du deny réglerait le second point.

### Accès distant à R1 (ACL 10)

```
access-list 10 permit 192.168.10.0 0.0.0.255
access-list 10 deny any
line vty 0 4
 access-class 10 in
 login local
 transport input ssh
```

`access-class` filtre les connexions vers le routeur lui-même, ce que les ACL d'interface ne font pas. Le `deny any` est là volontairement : le refus implicite à la fin d'une ACL ne compte pas les correspondances, un refus explicite si, donc une tentative bloquée apparaît dans `show access-lists 10`.

## Périmètre

J'ai gardé la sécurité de l'administration simple : SSH version 2, un utilisateur local, des clés RSA et `access-class`. Je n'ai pas mis en place de serveur AAA. CORE-SW n'a aucune configuration de management, ce que j'ai documenté comme une lacune plutôt que de laisser des réglages à moitié finis qui ressemblent à une protection.

## Problèmes et corrections

### ACL sans état et direction

Ce qui m'a pris le plus de temps, c'est de comprendre que les ACL sont sans état et que `in`/`out` se lit par rapport au routeur. C'est pour cela que l'ACL 100 est en entrée côté invités, et que les ACL 110 et 120 sont en sortie vers Finance et vers les serveurs.

### Configuration sur le mauvais routeur

J'ai créé la sous-interface `.40` et l'ACL 120 sur le routeur ISP par erreur. Le nettoyage :

```
Router(config)#interface gigabitEthernet 0/0.40
Router(config-subif)#no ip access-group 120 in
Router(config-subif)#exit
Router(config)#no access-list 120
Router(config)#no interface GigabitEthernet0/0.40
Router#copy running-config startup-config
```

La configuration finale du routeur ISP n'a plus de sous-interface ni d'ACL (`configs/isp-router.cfg`). Maintenant chaque équipement a son propre nom d'hôte, donc l'invite me dit où je suis.

### SSH fermé par l'hôte distant

```
C:\>ssh -l admin 192.168.10.1
[Connection to 192.168.10.1 closed by foreign host]
```

En lisant la configuration, j'ai vu `transport input ssh` et un utilisateur local, mais ni nom de domaine ni clés RSA, donc le routeur ne pouvait pas accepter de SSH. Après avoir généré les clés (section suivante), le test SSH depuis le PC IT a fonctionné (`screenshots/test-07-it-ssh-success.png`).

### Clés RSA refusées

La génération des clés a échoué deux fois. Cette version d'IOS a rejeté le mot-clé `modulus`, puis elle a refusé de générer les clés tant que le nom d'hôte était `Router` :

```
Router(config)#ip domain-name lab.local
Router(config)#crypto key generate rsa modulus 2048
% Invalid input detected at '^' marker.
Router(config)#crypto key generate rsa general-keys
% Please define a hostname other than Router.
Router(config)#hostname R1
R1(config)#crypto key generate rsa general-keys
The name for the keys will be: R1.lab.local
How many bits in the modulus [512]: 2048
% Generating 2048 bit RSA keys, keys will be non-exportable...[OK]
```

Le nom de la clé est construit à partir du nom d'hôte et du nom de domaine, donc les deux doivent être définis avant. L'ordre est : nom d'hôte, nom de domaine, clés, puis SSH. Capture : `screenshots/incidents/incident-rsa-hostname-error.png`.

### NAT sur la mauvaise interface

J'avais `ip nat inside` sur `Gi0/0`. Le trafic des VLAN arrive par les sous-interfaces, donc le marquage sur l'interface physique ne servait à rien pour lui. Je l'ai repéré en relisant la configuration exportée, pas à cause d'un test qui échouait. La correction :

```
R1(config)#interface gigabitEthernet 0/0
R1(config-if)#no ip nat inside
R1(config)#interface gigabitEthernet 0/0.10
R1(config-subif)#ip nat inside
! pareil sur 0/0.20, 0/0.30, 0/0.40 et 0/0.50
```

Ensuite, un ping d'un invité vers le routeur ISP est apparu dans `show ip nat translations` : 192.168.50.2 traduit en 203.0.113.2 (`screenshots/test-03-nat-translations.png`). Capture de la correction : `screenshots/incidents/incident-nat-subinterface-fix.png`.
