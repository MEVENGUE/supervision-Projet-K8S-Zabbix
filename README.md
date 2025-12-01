# Supervision du cluster Kubernetes avec Zabbix

> 🧩 **Suite du projet Kubernetes :**  
> Ce mini-projet de supervision s'appuie sur le cluster déjà déployé dans le dépôt :  
> 👉 https://github.com/MEVENGUE/K8S  

L'objectif est d'ajouter une **brique de supervision complète** autour du cluster Kubernetes en utilisant **Zabbix** :

- Supervision de l'hôte maître (`k8s-master`) et des nœuds (`k8s-workerX`)
- Détection d'anomalies (CPU / réseau / disponibilité des nœuds)
- Surveillance d'une application Kubernetes via un **scénario web**
- Production de tableaux de bord et de schémas pour le rapport

Tout est réalisé dans un contexte **Hyper-V** sur des VMs Ubuntu, en respectant les principes de cybersécurité (aucun mot de passe ou IP sensible n'apparaît dans le code).

---

## 1. Architecture globale

### 1.1. Topologie Hyper-V

- **Poste physique Windows**
  - Hyper-V activé
  - Accès à l'interface Zabbix via un navigateur (Firefox/Chrome)

- **VM 1 – `k8s-master`**
  - Ubuntu Server 22.04
  - Rôle :
    - Master du cluster Kubernetes (voir dépôt K8S)
    - **Zabbix Server**
    - **Base de données MySQL/MariaDB** dédiée à Zabbix
    - **Apache2 + frontend web Zabbix**
    - Agent Zabbix local

- **VM 2 – `k8s-worker1`**
  - Ubuntu Server 22.04
  - Nœud de travail Kubernetes
  - Agent Zabbix

- **VM 3 – `k8s-worker2`** (optionnel)
  - Même rôle que `k8s-worker1`

Les VMs sont sur le même réseau privé Hyper-V.  
Dans ce README, on utilisera des variables génériques :

- `IP_ZABBIX` : adresse IP de la VM `k8s-master`
- `ZABBIX_DB_PASSWORD` : mot de passe du compte MySQL `zabbix`
- `NODEPORT_APP` : NodePort exposant l'application de test (ex. 30080)


### 1.2. Schéma de l'architecture

![Diagramme de l'architecture Zabbix et Kubernetes](images/Diagramme%20Mermaid%20zabbix%20kubernetes.png)

### 1.3. Schéma réseau détaillé (Cisco Packet Tracer)

Le schéma suivant illustre la topologie réseau complète du projet avec les connexions entre les différents composants :

![Schéma Cisco Packet Tracer de fonctionnement du projet](images/schéma%20Cisco%20packet%20tracer%20de%20fonctionnement%20projet.jpg)

> **Note :** Le fichier source `.pkt` est disponible dans le dossier `Zabbix schéma Cisco/` pour une visualisation interactive dans Cisco Packet Tracer.

---

## 2. Prérequis

1. **Cluster Kubernetes fonctionnel** déployé selon le projet :  
   https://github.com/MEVENGUE/K8S

2. Accès à `k8s-master` en SSH avec un utilisateur sudo.

3. `kubectl` configuré sur `k8s-master` et capable de parler au cluster :

   ```bash
   kubectl get nodes
   ```

4. Ports ouverts **sur le réseau privé uniquement** :
   * 80/TCP (Apache Zabbix GUI)
   * 10051/TCP (Zabbix server)
   * 10050/TCP (Zabbix agent)

---

## 3. Installation de Zabbix sur `k8s-master`

### 3.1. Mise à jour du système

```bash
sudo apt update && sudo apt upgrade -y
```

### 3.2. Installation de MySQL / MariaDB

```bash
sudo apt install -y mysql-server
sudo systemctl enable --now mysql
```

Création de la base et de l'utilisateur Zabbix :

```bash
sudo mysql
```

```sql
CREATE DATABASE zabbix
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_bin;

CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'ZABBIX_DB_PASSWORD';

GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';

FLUSH PRIVILEGES;

EXIT;
```

> Remplace `ZABBIX_DB_PASSWORD` par un mot de passe fort **non publié**.

### 3.3. Ajout du dépôt Zabbix 6.4 (Ubuntu 22.04)

```bash
wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_6.4-1+ubuntu22.04_all.deb
sudo apt update
```

### 3.4. Installation des paquets Zabbix

```bash
sudo apt install -y \
  zabbix-server-mysql \
  zabbix-frontend-php \
  zabbix-apache-conf \
  zabbix-sql-scripts \
  zabbix-agent
```

### 3.5. Import du schéma SQL Zabbix

Zabbix crée des fonctions stockées ⇒ besoin d'autoriser les créateurs de fonctions :

```bash
sudo mysql -e "SET GLOBAL log_bin_trust_function_creators = 1;"
```

Import du schéma :

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | \
  mysql -uzabbix -p zabbix
```

(entrer `ZABBIX_DB_PASSWORD` lorsqu'il est demandé)

### 3.6. Configuration de `zabbix_server.conf`

```bash
sudo nano /etc/zabbix/zabbix_server.conf
```

Modifier les lignes suivantes :

```text
DBName=zabbix
DBUser=zabbix
DBPassword=ZABBIX_DB_PASSWORD
```

Sauvegarder puis redémarrer :

```bash
sudo systemctl restart zabbix-server zabbix-agent apache2
sudo systemctl enable zabbix-server zabbix-agent apache2
```

Vérification :

```bash
sudo systemctl status zabbix-server
sudo systemctl status zabbix-agent
```

---

## 4. Configuration de l'interface web Zabbix

Depuis le **PC Windows** :

1. Ouvrir un navigateur et accéder à :

   ```text
   http://IP_ZABBIX/zabbix
   ```

2. Suivre l'assistant d'installation :

   * Choisir la langue (FR)
   * Vérifier les prérequis PHP
   * Renseigner les paramètres MySQL :
     * Hôte : `localhost`
     * Base : `zabbix`
     * Utilisateur : `zabbix`
     * Mot de passe : `ZABBIX_DB_PASSWORD`
   * Nom du serveur Zabbix (ex : `Zabbix – Cluster K8s`)

3. Connexion avec le compte par défaut (à **modifier immédiatement**) :

   * Utilisateur : `Admin`
   * Mot de passe : `zabbix`

Changer le mot de passe de `Admin` et créer un compte perso pour l'usage quotidien.

---

## 5. Ajout des nœuds Kubernetes dans Zabbix

### 5.1. Installation de l'agent sur les workers

Sur **chaque worker** (`k8s-worker1`, `k8s-worker2`, etc.) :

```bash
sudo apt update
sudo apt install -y zabbix-agent
```

Éditer `/etc/zabbix/zabbix_agentd.conf` :

```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```

Adapter ces lignes :

```text
Server=IP_ZABBIX
ServerActive=IP_ZABBIX
Hostname=k8s-worker1   # ou k8s-worker2, etc.
```

Redémarrer l'agent :

```bash
sudo systemctl restart zabbix-agent
sudo systemctl enable zabbix-agent
```

### 5.2. Déclaration des hôtes dans l'interface

Dans la GUI Zabbix :

1. **Configuration → Hôtes → Créer un hôte**

2. Exemple pour `k8s-worker1` :

   * Nom : `k8s-worker1`
   * Groupes : `Kubernetes Cluster` (à créer si nécessaire)
   * Interfaces :
     * Type : Agent
     * IP : `IP_k8s_worker1`
     * Port : `10050`

3. Onglet **Modèles** :

   * Ajouter le modèle : `Linux by Zabbix agent`

4. Sauvegarder.

Répéter pour `k8s-workerX` et pour l'hôte `Zabbix server` (k8s-master) si ce n'est pas déjà fait.

![Ajout des nœuds worker1 et Worker 2 sur Zabbix](images/Ajout%20des%20noeuds%20worker1%20et%20Worker%202%20sur%20Zabbix.jpg)

---

## 6. Scénarios de supervision mis en place

### 6.1. Scénario 1 – Alerte CPU > 80% pendant 5 minutes

* **Hôte :** `Zabbix server`
* **Modèle utilisé :** `Linux by Zabbix agent`

Dans la GUI :

1. Aller dans **Configuration → Hôtes → Zabbix server → Déclencheurs**.

2. Créer un nouveau déclencheur :

   * Nom : `CPU: utilisation > 80% pendant 5 minutes`
   * Sévérité : `Moyen`

3. Expression (via le constructeur) :

   * Élément : `System: CPU utilization` (ou équivalent)
   * Fonction : `min(5m)`
   * Condition : `>`
   * Valeur : `80`

4. Enregistrer.

**Effet :** dès que la charge CPU moyenne > 80 % pendant 5 minutes, un problème apparaît dans **Surveiller → Problèmes**.

### 6.2. Scénario 2 – Nombre de nœuds Kubernetes non Ready

#### 6.2.1. Préparation de `kubectl` pour l'utilisateur `zabbix`

Sur `k8s-master` :

```bash
sudo mkdir -p /var/lib/zabbix/.kube
sudo cp /root/.kube/config /var/lib/zabbix/.kube/config
sudo chown -R zabbix:zabbix /var/lib/zabbix/.kube
sudo -u zabbix kubectl get nodes   # vérification
```

#### 6.2.2. Autoriser `system.run` dans l'agent (limité au labo)

Éditer `/etc/zabbix/zabbix_agentd.conf` sur `k8s-master` :

```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```

Ajouter/adapter :

```text
#DenyKey=system.run[*]
AllowKey=system.run[kubectl get nodes --no-headers | grep -v " Ready " | wc -l]
```

Puis :

```bash
sudo systemctl restart zabbix-agent
```

#### 6.2.3. Création de l'élément Zabbix

Dans la GUI → **Configuration → Hôtes → Zabbix server → Éléments → Créer** :

* Nom : `noeuds_k8s_not_ready`
* Type : `Agent Zabbix`
* Clé :

  ```text
  system.run[kubectl get nodes --no-headers | grep -v " Ready " | wc -l]
  ```

* Type d'information : `Numérique (entier non signé)`
* Intervalle d'actualisation : `60s`

Enregistrer.

![Élément de vérification pods créer sur zabbix avec valeur affiché](images/élément%20de%20vérification%20pods%20créer%20sur%20zabbix%20avec%20valeur%20affiché.jpg)

#### 6.2.4. Création du déclencheur

Toujours sur `Zabbix server` :

* Nom : `Cluster: au moins un nœud n'est pas Ready`
* Sévérité : `Haut`
* Expression :

  ```text
  {Zabbix server:noeuds_k8s_not_ready.last()}>0
  ```

![Déclencheur surveillance noeud et pods créer](images/déclencheur%20surveillance%20noeud%20et%20pods%20créer.jpg)

**Test :** arrêter le kubelet d'un worker ou éteindre la VM → le déclencheur passe en PROBLÈME.

![Commande stop noeud worker 2 et restart pour vérifier si le déclencheur fonctionne](images/commande%20stop%20noeud%20worker%202%20et%20restart%20pour%20vérifier%20si%20le%20déclencheur%20fonctionne.jpg)

![1 pods NotReady après commande](images/1%20pods%20NotReady%20après%20commande.jpg)

Lorsque tout repasse en `Ready`, le problème est résolu.

![Après quelques minutes problèmes résolu](images/Après%20quelques%20minutes%20problèmes%20résolu.jpg)

![Vérification si les noeuds sont tous Ready après commandes de restart](images/vérification%20si%20les%20noeuds%20sont%20tous%20Ready%20après%20commandes%20de%20restart.jpg)

### 6.3. Scénario 3 – Supervision d'une application Kubernetes via un scénario web

#### 6.3.1. Déploiement d'une appli de test dans le cluster

Sur `k8s-master` :

```bash
kubectl create deployment demo-nginx --image=nginx
kubectl expose deployment demo-nginx \
  --type=NodePort \
  --port=80 \
  --name=demo-nginx-service
kubectl get svc demo-nginx-service
```

Notez :

* `NODEPORT_APP` = NodePort affiché (ex. 30080)
* L'IP d'un nœud (ex. `IP_k8s-worker1`)

Test :

```bash
curl http://IP_k8s-worker1:NODEPORT_APP/
```

#### 6.3.2. Création du scénario web Zabbix

Dans la GUI Zabbix :

1. **Configuration → Hôtes → Zabbix server → Scénarios web → Créer un scénario**

2. Nom : `K8s_app_demo`

3. Intervalle : `1m`

Ajouter une étape :

* Nom : `page_principale`
* URL :

  ```text
  http://IP_k8s-worker1:NODEPORT_APP/
  ```

* Méthode : `GET`

Enregistrer.

#### 6.3.3. Déclencheur "application indisponible"

Sur `Zabbix server` → **Déclencheurs → Créer** :

* Nom : `K8s app demo: indisponible`
* Sévérité : `Moyen`
* Expression :

  ```text
  {Zabbix server:web.test.rspcode[K8s_app_demo,page_principale].last()}<>200
  ```

**Test :**

```bash
kubectl scale deployment demo-nginx --replicas=0
# ou kubectl delete deployment demo-nginx
```

L'alerte apparaît lorsque l'application n'est plus accessible.

![Déclencheurs créer dans Zabbix  et opérationnel](images/Déclencheurs%20créer%20dans%20Zabbix%20%20et%20opérationnel.jpg)

![Déclencheurs créer dans Zabbix  et opérationnel (2)](images/Déclencheurs%20créer%20dans%20Zabbix%20%20et%20opérationnel%20(2).jpg)

---

## 7. Tableaux de bord et visualisations

### 7.1. Dashboard "Kubernetes – Vue cluster"

Créé via **Surveiller → Tableaux de bord** :

* Widget **"Problèmes par sévérité"** filtré sur le groupe `Kubernetes Cluster`
* Widget **"Disponibilité des hôtes"** (status des VMs)
* Graphiques simples CPU / RAM par hôte
* Widget "Problèmes" filtré par tag `role=worker`

![Tableau créer dans Zabbix pour monitoring des noeuds](images/Tableau%20créer%20dans%20Zabbix%20pour%20monitoring%20des%20noeuds.jpg)

![Tableau créer dans Zabbix pour monitoring des noeuds partie 2](images/Tableau%20créer%20dans%20Zabbix%20pour%20monitoring%20des%20noeuds%20partie%202.jpg)

### 7.2. Graphiques de monitoring

Les graphiques permettent de visualiser l'évolution des métriques dans le temps :

![Graphique monitoring pods erreurs](images/Graphique%20monitoring%20pods%20erreurs.jpg)

Exemple de détection d'erreurs lorsque les pods ne sont pas disponibles :

![Pods en erreur car le projet fleetman pas lancé](images/pods%20en%20erreur%20car%20le%20projet%20fleetman%20pas%20lancé.jpg)

---

## 8. Bonnes pratiques de sécurité

* Ne jamais publier :
  * mots de passe (MySQL, Zabbix, comptes système),
  * clés privées,
  * IP publiques ou infos d'infra réelles.

* Restreindre autant que possible l'usage de `system.run` (clé `AllowKey`).

* Protéger l'accès à l'interface Zabbix (compte `Admin` désactivé ou mot de passe robuste).

* Zabbix doit idéalement être accessible **uniquement depuis le réseau d'admin interne**.

---

## 9. Pistes d'ajouts

* Ajout d'alertes par e-mail ou webhook (Slack, Teams, etc.)
* Création de modèles Zabbix spécifiques aux nœuds Kubernetes
* Monitoring approfondi des pods via l'API Kubernetes
* Intégration avec Grafana pour des dashboards avancés

---

Projet réalisé dans le cadre du mini-projet **"Kubernetes + Supervision Zabbix"**, en continuité du dépôt :

[https://github.com/MEVENGUE/K8S](https://github.com/MEVENGUE/K8S)

