# Compte rendu TP PetitPanier — Maryse / Rishbel / Fadil / Stella

Systèmes utilisés : Windows 11 + WSL2 (Ubuntu) + Docker Desktop

## Palier 1
1.1 : Le bloc Client correspond à l’outil Docker utilisé pour saisir les commandes.
Le bloc Server correspond au moteur Docker qui exécute les commandes et gère les conteneurs et les images. 
Si le bloc Server est absent, cela signifie que le client ne parvient pas à communiquer avec le moteur Docker, par exemple parce qu’il n’est pas démarré.

1.2 : Dans la commande -p 8080:80, le premier nombre 8080 correspond au port de la machine et le deuxième nombre 80 correspond au port du conteneur. 
Cette liaison permet d’accéder au serveur web du conteneur depuis le navigateur de la machine.

1.3 : Une image est un modèle qui contient les éléments nécessaires pour lancer une application. Un conteneur est une instance (un exemplaire) de cette image qui s’exécute.
 L’image peut rester présente même après l’arrêt et la suppression du conteneur.


## Palier 2
2.2 : Non, EXPOSE 8080 ne suffit pas pour accéder à la page depuis le navigateur. Cette instruction indique le port utilisé par l’application dans le conteneur,
 mais elle ne publie pas ce port sur la machine. Il faut utiliser -p avec docker run pour établir la liaison entre les deux ports.

2.3 : Le . à la fin de la commande indique que Docker utilise le dossier où l’on se trouve pour construire l’image. Il y cherche le Dockerfile et les fichiers nécessaires.
On utilise -p 8080:8080 car Nginx écoute sur le port 8080 dans le conteneur PetitPanier. Dans le premier palier, Nginx écoutait sur le port 80.

2.4 : Après la construction de l’image 2.0, la page affichait encore la version 1.0, parce que le conteneur en cours utilisait toujours l’ancienne image. 
Il faut supprimer ce conteneur et en lancer un nouveau à partir de l’image 2.0 pour afficher la nouvelle version sur le port 8080.


## Palier 3
3.2 : `Up` signifie que le conteneur tourne (le processus est lancé). `Up (healthy)` signifie qu'en plus le test de santé réussit, donc le service répond vraiment. 
C'est le bloc `healthcheck` du `compose.yaml` qui produit cette différence. Chez nous, le conteneur était `unhealthy` car `localhost` pointait vers l'IPv6 alors que nginx n'écoute qu'en IPv4. 
Preuve : `wget http://localhost:8080` donnait « Connection refused » alors que `wget http://127.0.0.1:8080` affichait la page. Correction : `127.0.0.1` dans le healthcheck, puis `Up (healthy)`.

3.3 : La seconde commande échoue car `localhost` désigne toujours la machine ou le conteneur depuis lequel on parle, ici busybox, qui n'héberge aucun site. 
`app` est le nom du service dans `compose.yaml` : Compose le traduit en adresse IP du conteneur grâce au DNS interne du réseau `petitpanier_default`.

## Palier 4
4.1 : `kind load` est nécessaire car le cluster kind tourne dans son propre conteneur, avec son propre stockage d'images : il ne voit pas les images de la machine hôte. 
En entreprise, on pousse l'image vers un registre (Docker Hub, GitHub Container Registry, registre privé) et Kubernetes la télécharge depuis là.

4.2 : Les labels `app: petitpanier` relient les objets : le Deployment sait quels Pods lui appartiennent, et le Service sait quels Pods viser. 
Si le `selector` du Service était différent, il ne trouverait aucun Pod : la liste ENDPOINTS serait vide et le site inaccessible.

4.3 : L'adresse dans ENDPOINTS (`10.244.0.5:8080`) est l'IP interne du Pod vers laquelle le Service redirige le trafic. 
Elle n'existe qu'à l'intérieur du cluster, donc on utilise `kubectl port-forward` pour créer un tunnel entre `localhost:8080` et le Service.

## Palier 5
5.1 : Les trois Pods peuvent tourner sur le même nœud, car le cluster local Kind ne comporte qu’un seul nœud de travail. On ne peut donc pas parler de haute disponibilité entre plusieurs nœuds.

5.2 : Le Deployment Kubernetes a recréé le Pod supprimé pour maintenir l’état désiré, c’est-à-dire conserver le nombre de Pods demandé.

5.3 : web est le nom du conteneur dans le Deployment. La commande kubectl set image permet de changer l’image utilisée.
kubectl rollout history affiche l'historique des révisions du Deployment. Après la mise à jour vers 2.0, il affichait 2 révisions (1 et 2). Après le rollout undo, il affiche 2 révisions (2 et 3) : l'ancienne révision 1 est recréée sous le numéro 3. 

## Palier 6 — tableau de diagnostic
| Panne   | Symptôme | Hypothèse | Commande de preuve et résultat | Correction |
|---|---|---|---|---|
| promo-1 | Le Pod ne démarre pas : ImagePullBackOff | L’image indiquée dans le Deployment ne peut pas être récupérée. | `kubectl describe pod promo-1-6bb5858789-r4dzb` : Events `Failed to pull image "petitpanier:1.O": repository does not exist` | Tag `1.O` (lettre O) remplacé par `1.0` (zéro), puis `kubectl apply -f pannes/panne-1.yaml` : Running, 1/1 |
| promo-2 | Le Pod affiche CrashLoopBackOff et redémarre. | La commande de démarrage échoue, car la variable PROMO_CODE est absente. | `kubectl logs promo-2-64cb6c59c9-tjxm5 --previous` : `ERREUR : variable PROMO_CODE absente` ; Events : `Back-off restarting failed container web` | Suppression de la ligne `command` (exit 1), puis `kubectl apply -f pannes/panne-2.yaml` : Running, 1/1 |
| promo-3 | Les Pods sont Running, mais le site ne répond pas. | Le Service ne cible aucun Pod : le selector ne correspond pas aux labels | `kubectl get endpoints promo-3` : `<none>` ; `kubectl get svc promo-3 -o yaml` : selector `app: promo-three` aucun endpoint n’est affiché avant correction. | Selector remplacé par `app: promo-3`, puis `kubectl apply -f pannes/panne-3.yaml` : 2 endpoints |


6.2 (--previous) : L’option --previous permet de consulter les logs du conteneur précédent, notamment après un redémarrage. Sans cette option, les logs peuvent être vides, car le conteneur actuel vient de redémarrer et n’a pas encore produit de message.

6.3 (pourquoi Running mais inaccessible) : Les Pods peuvent être Running, mais le Service ne peut pas leur transmettre les requêtes si son sélecteur ne correspond pas aux labels des Pods. Dans le cas de promo-3, le sélecteur app: promo-three ne correspond pas au label app: promo-3. Le Service n’a donc aucun endpoint.



## Palier 7
| Étape | Commande | Si cette étape échoue, que se passe-t-il ? |
|---|---|---|
| build | docker build -t petitpanier:1.0 . | L’image n’est pas construite. Il faut vérifier le Dockerfile et les fichiers nécessaires. |
| test | docker run --name petitpanier -d -p 8080:8080 petitpanier:1.0 | Le conteneur ne démarre pas ou la page ne s’affiche pas sur localhost:8080 |
| publish | kind load docker-image petitpanier:1.0 --name petitpanier | L’image n’est pas disponible dans le cluster Kind, pas de déploiement. |
| deploy | kubectl apply -f k8s/ puis kubectl rollout status deployment/petitpanier | Les ressources Kubernetes ne sont pas créées ou mises à jour. Il faut vérifier les fichiers YAML et les erreurs affichées. |

7.1 : Le pipeline utilise l’identifiant du commit comme tag afin d’identifier précisément la version du code ayant servi à construire l’image Docker. Contrairement au tag `latest`, qui peut désigner une image différente à chaque nouvelle construction, le tag associé au commit permet de retrouver et de déployer une version spécifique, ainsi que de revenir à une version antérieure en cas de problème.

Workflow n8n :
Webhook (le pipeline envoie le résultat du déploiement) → IF statut = failed → message Slack dans #petitpanier-ops indiquant que le déploiement de PetitPanier a échoué. 
Si statut = success, aucune alerte (ou un message de confirmation). Le token Slack est rangé dans les Credentials de n8n (coffre chiffré), pas dans le workflow ni dans le dépôt Git.


## Défis réalisés


Défi 1 : Il est impossible de publier deux conteneurs sur le même port `8081` de la machine hôte, car ce port réseau ne peut être écouté que par un seul processus à la fois. Lorsqu'on lance un conteneur avec l'option `-p 8081:80`, le moteur Docker réserve ce port sur la machine. Tenter d'y associer un second conteneur provoque un conflit de liaison. Il est donc indispensable d'utiliser des ports hôtes distincts (ex: `8081` et `8082`).

Défi 2 : Les étapes marquées `CACHED` correspondent au mécanisme d'optimisation du cache de construction (Build Cache) de Docker. Lorsque les instructions du `Dockerfile` et les fichiers source n'ont pas changé depuis la construction précédente, Docker réutilise instantanément les couches enregistrées au lieu de réexécuter les commandes, ce qui accélère considérablement le build.

Défi 3 : ajout d'un service `stats` (nginx:alpine, port 8082:80). La commande `docker compose exec stats wget -qO- http://app:8080` 
renvoie la page PetitPanier : les deux conteneurs partagent le réseau `petitpanier_default` et le DNS de Compose résout `app`.

Défi 4 : `kubectl describe pod` affiche 4 événements. **Scheduled** : le Pod est affecté au nœud petitpanier-control-plane. 
**Pulled** : « already present on machine », l'image est déjà sur le nœud grâce à `kind load`, donc pas de téléchargement. 
**Created** : le conteneur `web` est créé. **Started** : le conteneur démarre. Ensuite la readinessProbe réussit et le Pod passe `Ready`, ce qui permet au Service de lui envoyer du trafic.

Défi 6 : Modification du chemin de la readinessProbe dans k8s/deployment.yaml en remplaçant `/` par `/sante`. Le site web est immédiatement devenu `inaccessible` via le Service ou le port-forward, bien que les Pods affichent le statut Running. La commande `kubectl describe` pod a montré dans les Events que la sonde échouait en boucle avec une erreur 404 (HTTP probe failed with statuscode: 404), entraînant le retrait du Pod des endpoints. Le chemin a été rétabli à / pour que le site refonctionne normalement.   


Défi 7 :Rédaction du fichier `.gitlab-ci.yml` structuré en 4 jobs correspondants aux étapes du pipeline (build, test, publish, deploy).
