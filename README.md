------------------------------------------------------------------------------------------------------
ATELIER PRA/PCA
------------------------------------------------------------------------------------------------------
L’idée en 30 secondes : Cet atelier met en œuvre un **mini-PRA** sur **Kubernetes** en déployant une **application Flask** avec une **base SQLite** stockée sur un **volume persistant (PVC pra-data)** et des **sauvegardes automatiques réalisées chaque minute vers un second volume (PVC pra-backup)** via un **CronJob**. L’**image applicative est construite avec Packer** et le **déploiement orchestré avec Ansible**, tandis que Kubernetes assure la gestion des pods et de la disponibilité applicative. Nous observerons la différence entre **disponibilité** (recréation automatique des pods sans perte de données) et **reprise après sinistre** (perte volontaire du volume de données puis restauration depuis les backups), nous mesurerons concrètement les RTO et RPO, et comprendrons les limites d’un PRA local non répliqué. Cet atelier illustre de manière pratique les principes de continuité et de reprise d’activité, ainsi que le rôle respectif des conteneurs, du stockage persistant et des mécanismes de sauvegarde.
  
**Architecture cible :** Ci-dessous, voici l'architecture cible souhaitée.   
  
![Screenshot Actions](Architecture_cible.png)  
  
-------------------------------------------------------------------------------------------------------
Séquence 1 : Codespace de Github
-------------------------------------------------------------------------------------------------------
Objectif : Création d'un Codespace Github  
Difficulté : Très facile (~5 minutes)
-------------------------------------------------------------------------------------------------------
**Faites un Fork de ce projet**. Si besoin, voici une vidéo d'accompagnement pour vous aider à "Forker" un Repository Github : [Forker ce projet](https://youtu.be/p33-7XQ29zQ) 
  
Ensuite depuis l'onglet **[CODE]** de votre nouveau Repository, **ouvrez un Codespace Github**.
  
---------------------------------------------------
Séquence 2 : Création du votre environnement de travail
---------------------------------------------------
Objectif : Créer votre environnement de travail  
Difficulté : Simple (~10 minutes)
---------------------------------------------------
Vous allez dans cette séquence mettre en place un cluster Kubernetes K3d contenant un master et 2 workers, installer les logiciels Packer et Ansible. Depuis le terminal de votre Codespace copier/coller les codes ci-dessous étape par étape :  

**Création du cluster K3d**  
```
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```
```
k3d cluster create pra \
  --servers 1 \
  --agents 2
```
**vérification de la création de votre cluster Kubernetes**  
```
kubectl get nodes
```
**Installation du logiciel Packer (création d'images Docker)**  
```
PACKER_VERSION=1.11.2
curl -fsSL -o /tmp/packer.zip \
  "https://releases.hashicorp.com/packer/${PACKER_VERSION}/packer_${PACKER_VERSION}_linux_amd64.zip"
sudo unzip -o /tmp/packer.zip -d /usr/local/bin
rm -f /tmp/packer.zip
```
**Installation du logiciel Ansible**  
```
python3 -m pip install --user ansible kubernetes PyYAML jinja2
export PATH="$HOME/.local/bin:$PATH"
ansible-galaxy collection install kubernetes.core
```
  
---------------------------------------------------
Séquence 3 : Déploiement de l'infrastructure
---------------------------------------------------
Objectif : Déployer l'infrastructure sur le cluster Kubernetes
Difficulté : Facile (~15 minutes)
---------------------------------------------------  
Nous allons à présent déployer notre infrastructure sur Kubernetes. C'est à dire, créér l'image Docker de notre application Flask avec Packer, déposer l'image dans le cluster Kubernetes et enfin déployer l'infratructure avec Ansible (Création du pod, création des PVC et les scripts des sauvegardes aututomatiques).  

**Création de l'image Docker avec Packer**  
```
packer init .
packer build -var "image_tag=1.0" .
docker images | head
```
  
**Import de l'image Docker dans le cluster Kubernetes**  
```
k3d image import pra/flask-sqlite:1.0 -c pra
```
  
**Déploiment de l'infrastructure dans Kubernetes**  
```
ansible-playbook ansible/playbook.yml
```
  
**Forward du port 8080 qui est le port d'exposition de votre application Flask**  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
  
---------------------------------------------------  
**Réccupération de l'URL de votre application Flask**. Votre application Flask est déployée sur le cluster K3d. Pour obtenir votre URL cliquez sur l'onglet **[PORTS]** dans votre Codespace (à coté de Terminal) et rendez public votre port 8080 (Visibilité du port). Ouvrez l'URL dans votre navigateur et c'est terminé.  

**Les routes** à votre disposition sont les suivantes :  
1. https://...**/** affichera dans votre navigateur "Bonjour tout le monde !".
2. https://...**/health** pour voir l'état de santé de votre application.
3. https://...**/add?message=test** pour ajouter un message dans votre base de données SQLite.
4. https://...**/count** pour afficher le nombre de messages stockés dans votre base de données SQLite.
5. https://...**/consultation** pour afficher les messages stockés dans votre base de données.
  
---------------------------------------------------  
### Processus de sauvegarde de la BDD SQLite

Grâce à une tâche CRON déployée par Ansible sur le cluster Kubernetes (un CronJob), toutes les minutes une sauvegarde de la BDD SQLite est faite depuis le PVC pra-data vers le PCV pra-backup dans Kubernetes.  

Pour visualiser les sauvegardes périodiques déposées dans le PVC pra-backup, coller les commandes suivantes dans votre terminal Codespace :  

```
kubectl -n pra run debug-backup \
  --rm -it \
  --image=alpine \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "debug",
      "image": "alpine",
      "command": ["sh"],
      "stdin": true,
      "tty": true,
      "volumeMounts": [{
        "name": "backup",
        "mountPath": "/backup"
      }]
    }],
    "volumes": [{
      "name": "backup",
      "persistentVolumeClaim": {
        "claimName": "pra-backup"
      }
    }]
  }
}'
```
```
ls -lh /backup
```
**Pour sortir du cluster et revenir dans le terminal**
```
exit
```

---------------------------------------------------
Séquence 4 : 💥 Scénarios de crash possibles  
Difficulté : Facile (~30 minutes)
---------------------------------------------------
### 🎬 **Scénario 1 : PCA — Crash du pod**  
Nous allons dans ce scénario **détruire notre Pod Kubernetes**. Ceci simulera par exemple la supression d'un pod accidentellement, ou un pod qui crash, ou un pod redémarré, etc..

**Destruction du pod :** Ci-dessous, la cible de notre scénario   
  
![Screenshot Actions](scenario1.png)  

Nous perdons donc ici notre application mais pas notre base de données puisque celle-ci est déposée dans le PVC pra-data hors du pod.  

Copier/coller le code suivant dans votre terminal Codespace pour détruire votre pod :
```
kubectl -n pra get pods
```
Notez le nom de votre pod qui est différent pour tout le monde.  
Supprimez votre pod (pensez à remplacer <nom-du-pod-flask> par le nom de votre pod).  
Exemple : kubectl -n pra delete pod flask-7c4fd76955-abcde  
```
kubectl -n pra delete pod <nom-du-pod-flask>
```
**Vérification de la suppression de votre pod**
```
kubectl -n pra get pods
```
👉 **Le pod a été reconstruit sous un autre identifiant**.  
Forward du port 8080 du nouveau service  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
Observez le résultat en ligne  
https://...**/consultation** -> Vous n'avez perdu aucun message.
  
👉 Kubernetes gère tout seul : Aucun impact sur les données ou sur votre service (PVC conserve la DB et le pod est reconstruit automatiquement) -> **C'est du PCA**. Tout est automatique et il n'y a aucune rupture de service.
  
---------------------------------------------------
### 🎬 **Scénario 2 : PRA - Perte du PVC pra-data** 
Nous allons dans ce scénario **détruire notre PVC pra-data**. C'est à dire nous allons suprimer la base de données en production. Ceci simulera par exemple la corruption de la BDD SQLite, le disque du node perdu, une erreur humaine, etc. 💥 Impact : IL s'agit ici d'un impact important puisque **la BDD est perdue**.  

**Destruction du PVC pra-data :** Ci-dessous, la cible de notre scénario   
  
![Screenshot Actions](scenario2.png)  

🔥 **PHASE 1 — Simuler le sinistre (perte de la BDD de production)**  
Copier/coller le code suivant dans votre terminal Codespace pour détruire votre base de données :
```
kubectl -n pra scale deployment flask --replicas=0
```
```
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":true}}'
```
```
kubectl -n pra delete job --all
```
```
kubectl -n pra delete pvc pra-data
```
👉 Vous pouvez vérifier votre application en ligne, la base de données est détruite et la service n'est plus accéssible.  

✅ **PHASE 2 — Procédure de restauration**  
Recréer l’infrastructure avec un PVC pra-data vide.  
```
kubectl apply -f k8s/
```
Vérification de votre application en ligne.  
Forward du port 8080 du service pour tester l'application en ligne.  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
https://...**/count** -> =0.  
https://...**/consultation** Vous avez perdu tous vos messages.  

Retaurez votre BDD depuis le PVC Backup.  
```
kubectl apply -f pra/50-job-restore.yaml
```
👉 Vous pouvez vérifier votre application en ligne, **votre base de données a été restaureé** et tous vos messages sont bien présents.  

Relance des CRON de sauvgardes.  
```
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":false}}'
```
👉 Nous n'avons pas perdu de données mais Kubernetes ne gère pas la restauration tout seul. Nous avons du protéger nos données via des sauvegardes régulières (du PVC pra-data vers le PVC pra-backup). -> **C'est du PRA**. Il s'agit d'une stratégie de sauvegarde avec une procédure de restauration.  

---------------------------------------------------
Séquence 5 : Exercices  
Difficulté : Moyenne (~45 minutes)
---------------------------------------------------
**Complétez et documentez ce fichier README.md** pour répondre aux questions des exercices.  
Faites preuve de pédagogie et soyez clair dans vos explications et procedures de travail.  

**Exercice 1 :**  
Quels sont les composants dont la perte entraîne une perte de données ?  
  
*..Répondez à cet exercice ici..*

**Exercice 2 :**  
Expliquez nous pourquoi nous n'avons pas perdu les données lors de la supression du PVC pra-data  
  
*..Répondez à cet exercice ici..*

**Exercice 3 :**  
Quels sont les RTO et RPO de cette solution ?  
  
*..Répondez à cet exercice ici..*

**Exercice 4 :**  
Pourquoi cette solution (cet atelier) ne peux pas être utilisé dans un vrai environnement de production ? Que manque-t-il ?   
  
*..Répondez à cet exercice ici..*
  
**Exercice 5 :**  
Proposez une archtecture plus robuste.   
  
*..Répondez à cet exercice ici..*

---------------------------------------------------
Séquence 6 : Ateliers  
Difficulté : Moyenne (~2 heures)
---------------------------------------------------
### **Atelier 1 : Ajoutez une fonctionnalité à votre application**  
**Ajouter une route GET /status** dans votre application qui affiche en JSON :
* count : nombre d’événements en base
* last_backup_file : nom du dernier backup présent dans /backup
* backup_age_seconds : âge du dernier backup

*..**Déposez ici une copie d'écran** de votre réussite..*

---------------------------------------------------
### **Atelier 2 : Choisir notre point de restauration**  
Aujourd’hui nous restaurobs “le dernier backup”. Nous souhaitons **ajouter la capacité de choisir un point de restauration**.

*..Décrir ici votre procédure de restauration (votre runbook)..*  
  
---------------------------------------------------
Evaluation
---------------------------------------------------
Cet atelier PRA PCA, **noté sur 20 points**, est évalué sur la base du barème suivant :  
- Série d'exerices (5 points)
- Atelier N°1 - Ajout d'un fonctionnalité (4 points)
- Atelier N°2 - Choisir son point de restauration (4 points)
- Qualité du Readme (lisibilité, erreur, ...) (3 points)
- Processus travail (quantité de commits, cohérence globale, interventions externes, ...) (4 points) 



## Séquence 5 : Exercices

### Exercice 1 : Quels sont les composants dont la perte entraîne une perte de données ?

La perte de données survient uniquement si les **deux volumes persistants sont perdus simultanément** :

| Composant | Impact si perdu seul | Impact si perdu avec l'autre |
|---|---|---|
| **Pod Flask** | ❌ Aucune perte (stateless) | ❌ Aucune perte |
| **PVC pra-data** | ⚠️ Perte des données non encore sauvegardées (< 1 min) | 💥 Perte totale |
| **PVC pra-backup** | ⚠️ Perte de l'historique des backups | 💥 Perte totale |

Le Pod Flask est **stateless** : il ne stocke aucune donnée en lui-même.
Toute la donnée applicative réside dans le **PVC pra-data** (BDD en production).
Le **PVC pra-backup** contient les sauvegardes permettant la restauration.

> ⚠️ Point critique : les deux PVC résident sur le **même disque physique du node K3d**.
> Si ce disque est perdu, les deux PVC sont détruits simultanément → perte totale des données.

---

### Exercice 2 : Pourquoi nous n'avons pas perdu les données lors de la suppression du pod (Scénario 1 - PCA) ?

Dans Kubernetes, le **stockage est découplé du cycle de vie du pod**.

Un PVC (PersistentVolumeClaim) est un objet Kubernetes **indépendant** du pod.
Lorsque le pod Flask est supprimé :
1. Kubernetes détecte via le **Deployment** que le nombre de replicas souhaité (1) n'est plus atteint
2. Il recrée automatiquement un **nouveau pod** sous un nouvel identifiant
3. Ce nouveau pod monte le **même PVC pra-data**, qui n'a jamais été touché
4. La base SQLite est retrouvée intacte, aucune donnée n'est perdue

C'est la définition du **PCA (Plan de Continuité d'Activité)** :
> La disponibilité est assurée **automatiquement et sans intervention humaine**.
> Il n'y a aucune rupture de service et aucune perte de données.

---

### Exercice 3 : Quels sont les RTO et RPO de cette solution ?

#### RPO — Recovery Point Objective (Perte de données maximale acceptable)

> **RPO ≈ 1 minute**

Le CronJob de sauvegarde s'exécute **toutes les minutes**.
Dans le pire des cas, si un sinistre survient juste après un backup,
on perd au maximum **60 secondes** de données (les écritures non encore sauvegardées).

#### RTO — Recovery Time Objective (Durée de restauration maximale acceptable)

> **RTO ≈ 5 à 15 minutes** (manuel)

Le RTO se décompose ainsi :

| Étape | Durée estimée |
|---|---|
| Détection du sinistre | ~1-2 min |
| Suppression du pod et PVC corrompu | ~1 min |
| Recréation de l'infra (`kubectl apply`) | ~1-2 min |
| Lancement du job de restauration | ~1-2 min |
| Vérification et remise en service | ~1-2 min |
| **Total** | **~5 à 15 min** |

> ⚠️ Ce RTO est **manuel** et donc variable selon la disponibilité de l'opérateur.
> Il n'existe aucune automatisation de la procédure de reprise dans cet atelier.

---

### Exercice 4 : Pourquoi cette solution ne peut pas être utilisée en production ?

Cette architecture présente plusieurs **limitations critiques** qui la rendent inadaptée à un vrai environnement de production :

#### 🔴 Problèmes de résilience
- **Single Point of Failure** : les deux PVC sont sur le **même disque du node K3d**.
  Un crash du node entraîne la perte simultanée des données ET des backups.
- **Pas de réplication** : aucune copie des données dans un second datacenter ou zone de disponibilité.
  Un sinistre physique (incendie, inondation, panne matérielle) détruit tout.
- **Cluster K3d mono-node effectif** : K3d tourne dans un Codespace éphémère,
  si le Codespace est détruit, le cluster entier disparaît.

#### 🔴 Problèmes de sécurité
- **Backups non chiffrés** : les fichiers `.db` sont copiés en clair dans `/backup`.
- **Pas de contrôle d'accès** sur les volumes (RBAC insuffisant).
- **SQLite** n'est pas conçu pour un usage en production multi-utilisateurs
  (pas de connexions concurrentes, pas de haute disponibilité native).

#### 🔴 Problèmes opérationnels
- **Restauration 100% manuelle** : aucun runbook automatisé, dépendance à l'humain.
- **Pas de monitoring** : aucune alerte si le CronJob échoue ou si un backup est corrompu.
- **Pas de rétention** : les backups s'accumulent indéfiniment, sans rotation ni purge.
- **RPO de 1 minute** peut être insuffisant pour des données critiques (transactions bancaires, etc.).

---

### Exercice 5 : Proposition d'une architecture plus robuste

#### Améliorations proposées

**1. Base de données production-ready**
- Remplacer SQLite par **PostgreSQL** géré par un opérateur Kubernetes
  (ex : [CloudNativePG](https://cloudnative-pg.io/)) avec réplication synchrone master/replica.

**2. Stockage répliqué**
- Utiliser un **StorageClass avec réplication** entre nodes et zones :
  [Longhorn](https://longhorn.io/), [Rook/Ceph](https://rook.io/), ou le CSI natif du cloud provider.
- Activer les **VolumeSnapshots** CSI pour des snapshots instantanés et cohérents.

**3. Sauvegardes externes chiffrées**
- Déployer **[Velero](https://velero.io/)** pour sauvegarder namespaces + PVC
  vers un stockage objet externe (S3, Azure Blob, GCS) avec chiffrement AES-256.
- Politique de rétention : 7 jours de backups quotidiens, 4 semaines de backups hebdomadaires.

**4. Haute disponibilité du cluster**
- Cluster Kubernetes **multi-nœuds** avec 3 masters (etcd en HA)
- Déploiement **multi-zone** voire **multi-région** pour le DR géographique.
- `PodDisruptionBudgets` + `Liveness/Readiness probes` sur tous les pods critiques.

**5. Observabilité et automatisation**
- **Prometheus + Alertmanager** : alertes sur échec de CronJob, saturation du stockage.
- **Grafana** : tableaux de bord RTO/RPO en temps réel.
- **Runbook automatisé** de restauration via ArgoCD ou un opérateur custom.
- **Game Days** réguliers pour valider les RTO/RPO réels en conditions de production.

#### Comparaison RTO/RPO

| Architecture | RPO | RTO |
|---|---|---|
| Atelier (K3d + CronJob 1min) | ~1 minute | ~5-15 min (manuel) |
| Production (PostgreSQL + Velero) | ~0 (réplication sync) | ~2-5 min (automatisé) |
| Production multi-région | ~0 (réplication sync) | <1 min (bascule automatique) |
