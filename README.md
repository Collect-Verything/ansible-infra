
# 🚀 Infrastructure Automation avec Ansible & Microservice

## 📌 Contexte
Ce projet a pour objectif d’automatiser la configuration initiale et la sécurisation de serveurs grâce à **Ansible**.
Il part du principe qu’une batterie de serveurs a été préparée et qu’un fichier CSV contenant leurs adresses IP a été fourni, chaque serveur étant accessible via l’utilisateur root mais avec un mot de passe différent.
L’objectif est de préparer une série de serveurs prêts à être déployés pour des clients à une date encore incertaine, afin qu’ils puissent être immédiatement disponibles lorsque ceux-ci souhaiteront configurer et lancer leur boutique.L’idée est de disposer d’un **microservice** qui reçoit l’IP et le mot de passe initial root d’un serveur, génère dynamiquement l’inventaire Ansible (`inventory.ini`) et exécute un playbook (`setup.yml`) qui prépare le serveur pour la production.  

Une fois la première configuration appliquée, l’accès root par mot de passe est **désactivé**, et la connexion ne se fait plus que via un **utilisateur dédié (canse)** et des **clés SSH**.  

⚠️ Pour la suite du projet, les workflows GitHub Actions du project [produits-cms-ui](https://github.com/Collect-Verything/produits-cms-ui) devront être adaptés afin que les VM GitHub  puissent établir la connexion, puis se limiter uniquement au pull de l’image. Car actuellement elles font ce que ansible devrait faire ...
---

## ⚙️ Fonctionnalités du Playbook `setup.yml`
Lorsqu’un serveur est ajouté, Ansible applique les actions suivantes :

- ✅ **Mise à jour du système** (`apt update && apt upgrade`)  
- 🔒 **Installation et configuration UFW** (pare-feu)  
- 🛡️ **Installation de Fail2ban** (protection brute-force SSH)  
- 🐳 **Installation de Docker**  
- 👤 **Création de l’utilisateur `canse`** avec :  
  - mot de passe personnalisé (ex: `8641`)  
  - ajout dans les groupes `sudo` et `docker`  
  - clé publique SSH autorisée  
- 🚫 **Désactivation de l’accès root SSH**  
- 🗑️ **Suppression de la page par défaut IONOS** (`/var/www/html/index.html`)  

---

## 🔄 Workflow général

### Étapes Ansible
1. **Ajout d’un serveur** :  
   - via une requête API au microservice (`/add-server`)  
   - ou depuis une base de données contenant IP + mot de passe root.  

2. **Microservice** :  
   - régénère `inventory.ini` avec la nouvelle entrée (IP, mot de passe root)  
   - lance `ansible-playbook setup.yml`  

3. **Ansible** :  
   - applique la configuration complète sur le serveur cible  
   - installe les services de sécurité et déploie la clé SSH  

4. **Serveur prêt** :  
   - seul l’utilisateur `canse` peut se connecter  
   - root login est désactivé  
   - serveur sécurisé et prêt pour la production  


Parfait 👍 tu veux l’équivalent d’une section **Étapes Ansible**, mais cette fois pour ton **service** (le microservice + le futur frontend). Voici une version structurée que tu pourras coller directement dans ton README :

---

### Étapes Service

1. **Réception du fichier CSV**

    * via un **contrôleur API** du microservice (`/upload-csv`)
    * le fichier contient les adresses IP et les mots de passe root initiaux.

2. **Traitement et stockage**

    * le service lit le CSV,
    * crée automatiquement une **nouvelle table** dans la base de données (correspondant à une *range* de serveurs à configurer),
    * enregistre chaque serveur avec ses informations (IP, mot de passe root, état de configuration).

3. **Préparation Ansible**

    * le service écrit ou met à jour le fichier `inventory.ini` avec les nouveaux serveurs,
    * déclenche l’exécution du playbook `setup.yml`.

4. **Exécution Ansible**

    * Ansible applique la configuration et sécurise chaque serveur de la range,
    * installe les services (UFW, Fail2ban, Docker, etc.),
    * crée l’utilisateur `canse` avec clé SSH et nouveau mot de passe.

---

### Étapes Front


1**Consultation (Frontend)**

    * une interface permet de consulter toutes les **tables de ranges de serveurs** créées,
    * pour chaque serveur :

        * IP
        * mot de passe initial root
        * nouveau mot de passe `canse`
        * état de configuration (en attente, en cours, terminé).
        * selectionner un nouveau fichier csv pour une nouvelle range de server a configurer


---

## 📊 Schéma du workflow

### Version ASCII
```

+-----------+          +-------------------+         +----------------+
\|   Admin   |  --->    |   Microservice    |  --->   | inventory.ini  |
\|  (curl)   |          | (NestJS / FastAPI)|         |   (généré)     |
+-----------+          +-------------------+         +----------------+
\|                                |
\| lance ansible-playbook         |
v                                |
+-------------------+                   |
\|    Ansible CLI    | ------------------+
+-------------------+
|
v
+-----------------------+
\|   Serveur IONOS       |
\| - Upgrade + UFW       |
\| - Fail2ban + Docker   |
\| - User canse (SSH)    |
\| - Root login désactivé|
+-----------------------+

````

---

## 📂 Structure du projet

```
ansible-infra/
├── inventory.ini            
├── bootstrap-server            
    └── controller.ts
    └── service.ts
├── setup.yml                 
└── .github/
    └── workflows/
        └── deploy.yml       
```

---

## 🚀 Exemple d’utilisation API

### Requête pour ajouter un serveur

```bash
curl -X POST http://mon-microservice.local/add-server \
  -H "Content-Type: application/json" \
  -d '{
        "name": "serveur1",
        "ip": "1.2.3.4",
        "password": "motdepasseRoot"
      }'
```

### Réponse attendue

```json
{
  "status": "ok",
  "server": "serveur1"
}
```

---

## 🔐 Sécurité

* Les mots de passe root initiaux ne servent **qu’au premier bootstrap**.
* Une fois le playbook exécuté :
    * l’accès root est désactivé,
    * seul `canse` avec clé SSH peut se connecter.
* Les IP et mots de passe peuvent être stockés dans une **base de données sécurisée** (chiffrage recommandé).

---

## 🛠️ Améliorations futures

* Ajout d’une file de tâches pour le traitement asynchrone des playbooks.
* Passage complet à l’authentification SSH par clé (mot de passe root supprimé).
* Ajout de monitoring et alerting.

---

## 👨‍💻 Auteur

Projet conçu pour automatiser le **bootstrap de serveurs IONOS** avec **Ansible** + **Microservice personnalisé**.

```
