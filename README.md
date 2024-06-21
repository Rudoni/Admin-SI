# README - Admin SI

## Auteur
**RUDONI Antonin**  
**Date : 28/03/2024**

## Plan du Système d'Information

### Composition du Système d'Information
Le système d'information est composé des éléments suivants :
- Un serveur de base de données pour la comptabilité (répliqué)
- Un serveur NAS
- Un serveur Windows avec Active Directory (AD)
- 15 collaborateurs équipés d'ordinateurs portables
- Un serveur WEB
- 15 accès VPN (comptes AD)

### Adressage IP
Le réseau local est configuré avec l'adresse réseau suivante : `192.168.1.0/24`
- Serveur de base de données : `192.168.1.10/24`
- Serveur NAS : `192.168.1.11/24`
- Serveur Windows avec AD + DNS: `192.168.1.12/24`

Le réseau interne local est configuré avec l’adresse réseau suivante : `192.168.40.0/24`
- Adresses IP PC : de `192.168.40.21/24` à `192.168.1.34/24`

La DMZ est configurée avec l'adresse réseau suivante : `172.16.10.0/24`
- Serveur WEB : `172.16.10.1/24`

### Cartographie du Réseau
Le réseau est composé de 4 interfaces :
- **DMZ** : héberge le serveur web
- **WAN** : inclut un PC distant équipé de OpenVPN pour l’accès à la base de données locale
- **LAN** : divisé en deux segments pour des questions de sécurité (LAN et LAN interne)

Des mesures de sécurité incluent :
- Deux firewalls en redondance pour assurer la continuité de service
- Réplication des bases de données pour permettre un basculement rapide en cas de panne
- Un firewall pour isoler rapidement un poste compromis

## Règles du Firewall

| Nom          | Action | Interface   | Protocole | Source | Source Port | Destination   | Destination Port |
|--------------|--------|-------------|-----------|--------|-------------|---------------|------------------|
| WEB-HTTP     | Allow  | WAN         | TCP       | *      | *           | 192.168.1.9   | 80               |
| WEB-HTTPS    | Allow  | WAN         | TCP       | *      | *           | 192.168.1.9   | 443              |
| AD-RDP       | Allow  | LAN-INTERNE | TCP       | *      | *           | 192.168.1.12  | 389              |
| DB-SQL       | Allow  | LAN-INTERNE | TCP       | *      | *           | 192.168.1.9   | 3306             |
| VPN-OpenVPN  | Allow  | WAN         | UDP       | *      | *           | Passerelle    | 1194             |
| Sortant-Tous | Allow  | LAN         | TCP, UDP  | *      | *           | *             | *                |
| Entrant-Refus| Refuse | *           | TCP, UDP  | *      | *           | *             | *                |

## Plan de Reprise d'Activité (PRA) pour la Base de Données

1. **Sauvegarde quotidienne** de la base de données sur le serveur NAS.
2. **Réplication de la base de données** vers un serveur de secours hébergé dans un autre site.
3. **Serveur BDD répliqué** en local.
4. **Basculement automatique** vers le serveur de secours en cas de défaillance du serveur principal à l’aide d’un script.
5. Mise en place d'une **Garantie de Temps de Rétablissement (GTR)** de 1h et de 4h en cas de panne des 2 serveurs.
6. **Test régulier** du PRA pour s'assurer de son efficacité.
7. **Pare-feux répliqué** également avec le système CARP.

## Annexes

### Script de Sauvegarde (`bdd-backup.sh`)
```bash
#!/bin/bash
mysqldump -u root DATA > data.sql
scp data.sql root@192.168.1.10:~/
ssh 192.168.1.10 -c "cat data.sql | mysql -u root"
```

Ce fichier est enregistré dans `crontab` à ce chemin dans une machine Linux : `/etc/crontab`

### Tâches Planifiées
Pour les tâches planifiées, une fois par heure :
```
20 * * * * /etc/scripts/bdd-backup.sh
```
