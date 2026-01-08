# Projet pLPIC2 - Système de Contrôle Frontalier Numérique

## 🎯 Description du projet

Système automatisé de gestion et de contrôle des dossiers d'immigration sous Linux. Le système permet de recevoir, trier, valider et stocker automatiquement les archives déposées par les immigrés, tout en notifiant les inspecteurs pour validation finale.

**Contexte** : Numérisation du processus de contrôle des frontières pour gérer efficacement les problèmes d'immigration.

## 💻 Architecture technique

Le système repose sur **2 machines virtuelles Linux** :

### Machine 1 : Router-Grestin (Cerveau réseau et sécurité)
- **Rôle** : Routeur, serveur DNS, serveur DHCP, pare-feu
- **IP** : 192.168.10.1
- **Domaine** : grestin.gov
- **Zone** : DMZ (Zone Démilitarisée)

### Machine 2 : Server-Internal (Stockage et automatisation)
- **Rôle** : Serveur de fichiers sécurisé, messagerie, scripts d'automatisation
- **IP** : 192.168.10.2
- **Réseau** : LAN interne (192.168.10.0/24)

## 📦 Fonctionnement du système

### Flux des archives
1. Les immigrés déposent une archive (ZIP, etc.) sur le serveur externe
2. Chaque archive contient :
   - Un fichier TXT avec les informations personnelles (pays, raison de visite, durée, taille, poids, etc.)
   - Des documents PDF avec les numéros d'identification
3. Le système traite automatiquement les archives et les classe dans 3 dossiers :
   - **inbound** : Fichiers en attente de traitement
   - **classified** : Dossiers acceptés
   - **rejected** : Dossiers refusés

### Critères de validation
- Présence de mots-clés spécifiques dans les fichiers
- Structure conforme de l'archive
- Documents requis présents

## 🔧 Services implémentés

### Router-Grestin (Machine 1)

**Pare-feu (UFW)**
- Sécurisation du réseau en bloquant tout trafic non essentiel
- Seuls les services nécessaires sont autorisés : DNS, DHCP, Mail, Samba

**DNS (Bind9)**
- Permet la résolution de noms de domaine
- Domaine : server-internal.grestin.gov → 192.168.10.2
- Zone configurée : grestin.gov

**DHCP (ISC DHCP Server)**
- Distribution automatique des adresses IP aux clients
- Plage : 192.168.10.100 à 192.168.10.200
- DNS pointant vers le routeur (192.168.10.1)

### Server-Internal (Machine 2)

**Redondance des données (RAID 1)**
- Configuration RAID 1 avec mdadm
- Volume monté sur /srv/files
- Protection contre les pannes de disque

**Partage de fichiers (Samba)**
- Permet aux inspecteurs de se connecter et d'accéder au volume RAID
- Point de montage : /srv/files
- Utilisateurs créés : inspector1

**Messagerie (Postfix)**
- Envoi de notifications par email
- Domaine : grestin.gov
- Alertes système et confirmations de statut aux inspecteurs

**Automatisation (Cron)**
- Scripts planifiés pour s'exécuter automatiquement

## 🤖 Scripts d'automatisation

### Script de transfert (transfer.sh)
- **Fréquence** : Toutes les 15 minutes
- **Fonction** : Déplace les archives du dossier d'entrée vers les dossiers de classification
- **Logique** : Recherche de mots-clés spécifiques (ex: "Arms" pour rejeter)
- **Actions** :
  - Archives conformes → dossier "classified"
  - Archives non conformes → dossier "rejected"
  - Journalisation de toutes les opérations

### Script de nettoyage (cleanup.sh)
- **Fréquence** : Quotidienne (minuit)
- **Fonction** : Maintenance du volume RAID
- **Actions** :
  - Suppression des fichiers de plus de 10 jours
  - Suppression des répertoires vides

### Script de recherche (find_case.sh)
- **Fonction** : Outil pour les inspecteurs
- **Usage** : Permet de vérifier rapidement le statut d'un dossier
- **Statuts possibles** :
  - ACCEPTED : Dossier dans /classified
  - REJECTED : Dossier dans /rejected
  - PENDING : Dossier dans /inbound
  - INCONNU : Dossier non trouvé

## 👮 Poste des inspecteurs

Les inspecteurs disposent de :
- **Accès direct** au serveur interne via Samba
- **Programme de gestion** pour ouvrir et examiner les documents
- **Capacité de décision** : Accepter ou refuser les dossiers
- **Messagerie interne** : Email/IMAP pour les notifications
- **Accès Internet restreint** : Uniquement vers les sites gouvernementaux
- **Outil de recherche** : Script find_case.sh pour vérifier les identifiants

## 🔒 Sécurité

**Architecture en zones**
- Zone DMZ pour le serveur de fichiers externe
- Réseau interne (LAN) isolé et sécurisé
- Routeur faisant office de pare-feu entre les zones

**Pare-feu (UFW)**
- Politique par défaut : DENY (tout est bloqué)
- Ouverture sélective des ports :
  - DNS (53/udp)
  - DHCP (67-68/udp)
  - SMTP (25/tcp)
  - Samba (445/tcp)
  - SSH (22/tcp)

**RAID 1**
- Protection contre la perte de données
- Redondance en cas de panne d'un disque

## ✅ Tests et validations

Tous les composants ont été testés et validés :
- ✓ Connectivité réseau entre les machines
- ✓ Distribution DHCP fonctionnelle
- ✓ Résolution DNS opérationnelle
- ✓ Pare-feu actif et configuré
- ✓ Volume RAID monté et accessible
- ✓ Service Samba actif
- ✓ Configuration Postfix validée
- ✓ Scripts cron planifiés
- ✓ Logique de flux et de classification testée

## 📊 Architecture réseau

```
INTERNET
    ↓
Router-Grestin (192.168.10.1)
- UFW Firewall
- DHCP Server
- DNS Server (Bind9)
    ↓
    ├── DMZ : External File Server (192.168.20.2)
    │   - Dépôt des archives
    │   - Scripts de transfert
    │
    └── LAN Interne (192.168.10.0/24)
        └── Server-Internal (192.168.10.2)
            - Stockage RAID 1
            - Samba/NFS
            - Postfix
            - Scripts d'automatisation
            └── Postes Inspecteurs (192.168.10.100+)
                - Accès partages réseau
                - Email (SMTP/IMAP)
                - Outils de validation
```

## 📝 Informations pratiques

- **Réseau** : 192.168.10.0/24
- **Domaine** : grestin.gov
- **Utilisateurs** : inspector1
- **Dossiers** : /srv/files/{inbound, classified, rejected}
- **Logs** : /var/log/transfer.log

---

**Projet** : pLPIC2  
**Objectif** : Sécurité et Redondance  
**Statut** : Opérationnel
