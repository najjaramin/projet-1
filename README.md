# Mini-projet Kubernetes : Déploiement d'une application Web et API

## 1. Présentation du projet

Ce projet a pour objectif de déployer et de tester une application composée d'un frontend Web et d'une API REST dans un cluster Kubernetes à trois nœuds.

Le projet permet de mettre en pratique les concepts suivants :

- Le déploiement d'applications avec Kubernetes.
- La gestion des Pods avec les Deployments.
- L'exposition des applications avec les Services.
- Le routage HTTP avec Ingress et Traefik.
- La communication interne entre les composants.
- La vérification du fonctionnement et le diagnostic des erreurs.

## 2. Architecture technique

Le cluster Kubernetes est installé dans des machines virtuelles VMware utilisant un réseau NAT.

| Machine | Adresse IP | Rôle |
|---|---|---|
| master | 192.168.30.172 | Nœud de contrôle et administration |
| w1 | 192.168.30.171 | Nœud de travail |
| w2 | 192.168.30.173 | Nœud de travail |

### Composants déployés

- **Kubernetes / kubeadm** : orchestration des conteneurs.
- **containerd** : runtime des conteneurs.
- **Traefik** : contrôleur Ingress pour le routage HTTP.
- **Frontend** : serveur Web NGINX, image `nginx:1.26-alpine`.
- **API** : service Podinfo, image `ghcr.io/stefanprodan/podinfo:6.5.4`.

### Répartition des ressources

| Ressource | Nom | Configuration |
|---|---|---|
| Namespace | `mini-projet` | Isolation logique du projet |
| Deployment | `frontend` | 2 réplicas |
| Deployment | `api` | 3 réplicas |
| Service | `frontend-svc` | Port 80 vers le frontend |
| Service | `api-svc` | Port 80 vers le port 9898 de l'API |
| Ingress | `mini-projet-ingress` | Routage selon le nom d'hôte |

## 3. Structure du dépôt

```text
k8s-mini-projet/
├── README.md
├── patch-ingress.json
└── manifests/
    ├── 00-namespace.yaml
    ├── 01-api.yaml
    ├── 02-frontend.yaml
    ├── 03-services.yaml
    └── 04-ingress.yaml
```

### Description des fichiers

- `00-namespace.yaml` : crée le namespace `mini-projet`.
- `01-api.yaml` : définit le Deployment de l'API et ses réplicas.
- `02-frontend.yaml` : définit le Deployment du frontend NGINX.
- `03-services.yaml` : crée les Services internes du frontend et de l'API.
- `04-ingress.yaml` : configure le routage HTTP avec Traefik.
- `patch-ingress.json` : fichier de patch associé à la configuration Ingress.

## 4. Prérequis

Avant de déployer le projet, vérifier que :

- Le cluster Kubernetes est opérationnel.
- `kubectl` est configuré sur la machine Master.
- Les trois nœuds sont accessibles et à l'état `Ready`.
- Traefik est installé dans le namespace `traefik`.
- Les images des applications peuvent être téléchargées.

Vérifier l'état du cluster :

```bash
kubectl get nodes -o wide
kubectl get pods -n traefik
kubectl cluster-info
```

## 5. Déploiement de l'application

Toutes les commandes suivantes sont exécutées depuis le terminal **Ubuntu Master**, dans le dossier du projet.

### Étape 1 : accéder au projet

```bash
cd ~/k8s-mini-projet
```

### Étape 2 : appliquer les manifests Kubernetes

```bash
kubectl apply -f manifests/
```

Cette commande crée ou met à jour les ressources déclarées dans les fichiers YAML.

### Étape 3 : vérifier les Deployments

```bash
kubectl get deployments -n mini-projet
```

Résultat attendu :

- `frontend` : 2 réplicas disponibles sur 2.
- `api` : 3 réplicas disponibles sur 3.

### Étape 4 : vérifier les Pods

```bash
kubectl get pods -n mini-projet -o wide
```

Les cinq Pods doivent être à l'état `Running` et être prêts à recevoir du trafic.

### Étape 5 : vérifier les Services

```bash
kubectl get services -n mini-projet
kubectl get endpoints -n mini-projet
```

Les Services doivent posséder des points de terminaison correspondant aux Pods disponibles.

## 6. Configuration du routage avec Traefik

Le contrôleur Traefik utilise le port NodePort HTTP `30080` pour accéder à l'application depuis le réseau du laboratoire.

Les noms d'hôtes configurés sont :

| Nom d'hôte | Service cible | Port |
|---|---|---|
| `web.k3s.local` | `frontend-svc` | 80 |
| `api.k3s.local` | `api-svc` | 80 |

Le port HTTP est accessible sur les nœuds Kubernetes via le NodePort `30080`.

Vérifier l'Ingress :

```bash
kubectl get ingress -n mini-projet
kubectl describe ingress mini-projet-ingress -n mini-projet
```

## 7. Tests de fonctionnement

### Test 1 : vérifier la communication interne avec l'API

```bash
kubectl exec -n mini-projet deployment/frontend -- wget -qO- http://api-svc:80/
```

Résultat attendu : une réponse JSON de Podinfo indiquant notamment la version `6.5.4`.

Ce test vérifie que le frontend peut communiquer avec l'API à travers le Service Kubernetes.

### Test 2 : vérifier le frontend NGINX

```bash
kubectl exec -n mini-projet deployment/frontend -- wget -qO- http://frontend-svc:80/
```

Résultat attendu : le code HTML de la page d'accueil NGINX.

### Test 3 : tester l'API à travers Traefik

```bash
curl -i -H "Host: api.k3s.local" http://192.168.30.172:30080/
```

Résultat attendu : une réponse HTTP `200 OK` contenant le JSON de Podinfo.

### Test 4 : tester le frontend à travers Traefik

```bash
curl -i -H "Host: web.k3s.local" http://192.168.30.172:30080/
```

Résultat attendu : une réponse HTTP `200 OK` contenant le HTML de NGINX.

**Remarque :** ces tests utilisent l'en-tête HTTP `Host` pour sélectionner la règle Ingress. Ils ne nécessitent donc pas de configuration DNS locale.

## 8. Diagnostic et dépannage

### Vérifier les ressources

```bash
kubectl get all -n mini-projet
kubectl get ingress -n mini-projet
kubectl get pods -n traefik
```

### Examiner les événements

```bash
kubectl get events -n mini-projet --sort-by=.metadata.creationTimestamp
```

### Examiner les logs de l'API

```bash
kubectl logs -n mini-projet deployment/api
```

### Examiner les logs du frontend

```bash
kubectl logs -n mini-projet deployment/frontend
```

### Vérifier les points de terminaison

```bash
kubectl get endpoints api-svc frontend-svc -n mini-projet
```

Si un Service ne possède aucun endpoint, vérifier son sélecteur et les labels des Pods correspondants.

### Exemple de panne et de récupération

Une modification incorrecte du sélecteur de `api-svc` peut supprimer ses endpoints et provoquer une erreur HTTP `503 Service Unavailable` à travers Traefik.

Pour rétablir le sélecteur attendu :

```bash
kubectl patch service api-svc -n mini-projet --type=merge --patch "spec:
  selector:
    app: api
    tier: backend"
```

Vérifier ensuite le retour des endpoints :

```bash
kubectl get endpoints api-svc -n mini-projet
```

Puis retester l'API :

```bash
curl -i -H "Host: api.k3s.local" http://192.168.30.172:30080/
```

Cette correction suppose que les labels du Deployment API sont bien `app: api` et `tier: backend`.

## 9. Commandes utiles pour l'administration

Afficher toutes les ressources du projet :

```bash
kubectl get all -n mini-projet
```

Suivre l'état des Pods :

```bash
kubectl get pods -n mini-projet -w
```

Consulter l'historique du Deployment API :

```bash
kubectl rollout history deployment/api -n mini-projet
```

Redémarrer le Deployment API si nécessaire :

```bash
kubectl rollout restart deployment/api -n mini-projet
```

Supprimer les ressources du projet lorsqu'elles ne sont plus nécessaires :

```bash
kubectl delete -f manifests/
```

**Attention :** cette dernière commande supprime les ressources déclarées dans les manifests. Vérifier le contenu des fichiers avant de l'exécuter.

## 10. Résultats obtenus

Les tests réalisés ont confirmé les points suivants :

- Les trois nœuds Kubernetes sont à l'état `Ready`.
- Le Deployment API fonctionne avec 3 réplicas.
- Le Deployment frontend fonctionne avec 2 réplicas.
- Les Services disposent de leurs endpoints.
- La communication interne entre le frontend et l'API fonctionne.
- Le frontend NGINX répond correctement.
- Le routage HTTP par Traefik fonctionne pour les deux noms d'hôtes.
- Les requêtes testées vers l'API renvoient une réponse HTTP `200 OK`.
- Une panne liée au sélecteur du Service API a été simulée puis corrigée.

Ces résultats valident le fonctionnement des composants testés. Ils ne constituent pas une mesure formelle de disponibilité sans interruption pendant une mise à jour.

## 11. Conclusion

Ce mini-projet met en pratique le déploiement d'une application Web et d'une API sur Kubernetes. Il démontre l'utilisation des Deployments, des Services, des namespaces et d'Ingress avec Traefik.

Les tests de communication, de routage et de dépannage permettent de mieux comprendre le fonctionnement d'une architecture conteneurisée distribuée sur plusieurs nœuds.

---

**Technologies :** Kubernetes, kubeadm, containerd, VMware, Traefik, NGINX, Podinfo et YAML.

**Dossier du projet sur la machine Master :** `~/k8s-mini-projet`
