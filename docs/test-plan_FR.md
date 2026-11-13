# Plan de test et résultats

[English](test-plan.md) | **Français**

Chaque test a été lancé depuis l'invite de commandes d'un PC (ou le navigateur web pour HTTP). Pour les cas bloqués, j'ai aussi regardé les compteurs de l'ACL sur R1.

| # | De -> vers | Test | Attendu | Observé | Preuve |
|---|---|---|---|---|---|
| 1 | Invité -> Finance | `ping 192.168.30.2` | Bloqué | 4 paquets perdus sur 4, « Destination host unreachable » depuis 192.168.50.1 | `test-01-02-guest-blocked.png` |
| 2 | Invité -> Serveurs | `ping 192.168.40.2` | Bloqué | 4 perdus sur 4, inaccessible depuis 192.168.50.1 | `test-01-02-guest-blocked.png` |
| 3 | Invité -> Internet | `ping 203.0.113.1` | Autorisé | 4 réponses sur 4, TTL 254. La table NAT montre 192.168.50.2 -> 203.0.113.2 | `test-03-guest-internet-ping.png`, `test-03-nat-translations.png` |
| 4 | RH -> Finance | `ping 192.168.30.2` | Bloqué | 4 perdus sur 4, inaccessible depuis 192.168.20.1 | `test-04-hr-to-finance-blocked.png` |
| 5 | RH -> Serveurs | navigateur, `http://192.168.40.2` | Autorisé | La page s'affiche. Ligne `eq www` de l'ACL 120 : 5 correspondances | `test-05-hr-browser-http.png`, `acl120-counters.png` |
| 6 | RH -> Serveurs | `ping 192.168.40.2` et `.40.3` | Bloqué | 8 perdus sur 8, inaccessible depuis 192.168.20.1. Ligne deny de l'ACL 120 : 8 correspondances | `test-06-hr-to-servers-blocked.png`, `acl120-counters.png` |
| 7 | IT -> R1 | `ssh -l admin 192.168.10.1` | Autorisé | Invite de mot de passe, puis l'invite `R1#` | `test-07-it-ssh-success.png` |
| 8 | RH -> R1 | `ssh -l admin 192.168.20.1` | Bloqué | « Connection refused by remote host ». `deny any` de l'ACL 10 : 5 correspondances | `test-08-hr-ssh-refused.png`, `test-08-acl10-counters.png` |
| 9 | IT -> Finance | `ping 192.168.30.2` | Autorisé | 3 réponses sur 4 (le premier paquet est perdu à cause d'ARP) | `test-09-10-it-ping-ipconfig.png` |
| 10 | Un PC par VLAN | `ipconfig` | Adresse de son propre VLAN | Voir le tableau ci-dessous | `test-09-10-it-ping-ipconfig.png` |

Le premier ping d'un test expire souvent parce que le PC doit d'abord résoudre la passerelle avec ARP. Seuls les paquets suivants comptent.

## Adresses distribuées par le DHCP

| VLAN | Hôte | Adresse | Passerelle |
|---|---|---|---|
| 10 IT | PC IT | 192.168.10.3 | 192.168.10.1 |
| 20 RH | PC RH | 192.168.20.2 | 192.168.20.1 |
| 30 Finance | PC Finance | 192.168.30.3 | 192.168.30.1 |
| 40 Serveurs | PC Serveurs | 192.168.40.3 | 192.168.40.1 |
| 50 Invités | PC Invité | 192.168.50.2 | 192.168.50.1 |

## Autres vérifications

- Les clés RSA existent sur R1 : `design-rsa-keys-r1.png` (`show crypto key mypubkey rsa`).
- Le serveur web a HTTP activé : `design-server-http-service.png`.

## Un résultat que j'ai dû refaire

Ma première exécution du test 6 a montré des réponses de 192.168.40.3, qui aurait dû être bloqué. J'ai remis les compteurs à zéro, relancé les deux pings dans la même session, et les 8 ont été bloqués et comptés. Je pense que la première exécution a eu lieu avant l'application de l'ACL, mais je ne l'ai pas confirmé.
