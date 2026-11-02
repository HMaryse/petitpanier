# Compte rendu TP PetitPanier — Maryse / Rishbel / Fadil / Stella

Systèmes utilisés : Windows 11 + WSL2 (Ubuntu) + Docker Desktop

## Palier 1
1.1 :
1.2 :
1.3 :

## Palier 2
2.2 :
2.3 :
2.4 :

## Palier 3
3.2 : `Up` signifie que le conteneur tourne (le processus est lancé). `Up (healthy)` signifie qu'en plus le test de santé réussit, donc le service répond vraiment. C'est le bloc `healthcheck` du `compose.yaml` qui produit cette différence. Chez nous, le conteneur était `unhealthy` car `localhost` pointait vers l'IPv6 alors que nginx n'écoute qu'en IPv4. Preuve : `wget http://localhost:8080` donnait « Connection refused » alors que `wget http://127.0.0.1:8080` affichait la page. Correction : `127.0.0.1` dans le healthcheck, puis `Up (healthy)`.

3.3 : La seconde commande échoue car `localhost` désigne toujours la machine ou le conteneur depuis lequel on parle, ici busybox, qui n'héberge aucun site. `app` est le nom du service dans `compose.yaml` : Compose le traduit en adresse IP du conteneur grâce au DNS interne du réseau `petitpanier_default`.

## Palier 4
4.1 : `kind load` est nécessaire car le cluster kind tourne dans son propre conteneur, avec son propre stockage d'images : il ne voit pas les images de la machine hôte. En entreprise, on pousse l'image vers un registre (Docker Hub, GitHub Container Registry, registre privé) et Kubernetes la télécharge depuis là.

4.2 : Les labels `app: petitpanier` relient les objets : le Deployment sait quels Pods lui appartiennent, et le Service sait quels Pods viser. Si le `selector` du Service était différent, il ne trouverait aucun Pod : la liste ENDPOINTS serait vide et le site inaccessible.

4.3 : L'adresse dans ENDPOINTS (`10.244.0.5:8080`) est l'IP interne du Pod vers laquelle le Service redirige le trafic. Elle n'existe qu'à l'intérieur du cluster, donc on utilise `kubectl port-forward` pour créer un tunnel entre `localhost:8080` et le Service.

## Palier 5
5.1 :
5.2 :
5.3 :

## Palier 6 — tableau de diagnostic
| Panne | Symptôme | Hypothèse | Commande de preuve et résultat | Correction |
|---|---|---|---|---|
| promo-1 | | | | |
| promo-2 | | | | |
| promo-3 | | | | |

6.2 (--previous) :
6.3 (pourquoi Running mais inaccessible) :

## Palier 7
| Étape | Commande | Si échec |
|---|---|---|
| build | | |
| test | | |
| publish | | |
| deploy | | |

7.1 :
Workflow n8n :

## Défis réalisés
Défi 3 : ajout d'un service `stats` (nginx:alpine, port 8082:80). La commande `docker compose exec stats wget -qO- http://app:8080` 
renvoie la page PetitPanier : les deux conteneurs partagent le réseau `petitpanier_default` et le DNS de Compose résout `app`.

Défi 4 : `kubectl describe pod` affiche 4 événements. **Scheduled** : le Pod est affecté au nœud petitpanier-control-plane. 
**Pulled** : « already present on machine », l'image est déjà sur le nœud grâce à `kind load`, donc pas de téléchargement. 
**Created** : le conteneur `web` est créé. **Started** : le conteneur démarre. Ensuite la readinessProbe réussit et le Pod passe `Ready`, ce qui permet au Service de lui envoyer du trafic.
