# TP Kubernetes - Bijouterie

Ce fichier donne l'ordre des operations. Le code de l'application fourni n'a pas ete modifie.

## 1. Valeurs personnelles a remplacer

Avant de publier le depot :

- remplacer `votre-user-dockerhub` dans le `Jenkinsfile` et les trois Deployments ;
- remplacer `repoURL` dans `argocd/application.yaml` ;
- choisir d'autres mots de passe, les encoder en base64 puis remplacer les valeurs dans les deux Secrets ;
- mettre les memes identifiants dans `mysql-secret` et `backend-secret`.

## 2. Test avec Docker Compose

```bash
docker compose up --build -d
docker compose ps
curl http://localhost:8080/api/config
docker compose down
```

## 3. Test Kubernetes avec kind

```bash
kind create cluster --name bijouterie --config kind-config.yaml

docker build -t votre-user-dockerhub/bijouterie-frontend:1 frontend/
docker build -t votre-user-dockerhub/bijouterie-backend:1 backend/
docker build -t votre-user-dockerhub/bijouterie-mysql:1 database/

kind load docker-image votre-user-dockerhub/bijouterie-frontend:1 --name bijouterie
kind load docker-image votre-user-dockerhub/bijouterie-backend:1 --name bijouterie
kind load docker-image votre-user-dockerhub/bijouterie-mysql:1 --name bijouterie

kubectl apply -f k8s/
kubectl get all -n bijouterie
kubectl get configmap,secret -n bijouterie
kubectl get endpoints -n bijouterie
curl http://localhost:30080/api/config
```

Apres la verification locale, Argo CD devient la seule source de verite. Il ne faut alors plus utiliser `kubectl apply` dans le namespace `bijouterie`.

## 4. Jenkins

1. Creer les trois depots publics sur Docker Hub.
2. Ajouter dans Jenkins un credential `Username with password` nomme `dockerhub-credentials`. Le password est le token Docker Hub.
3. Creer un job `Pipeline script from SCM`, indiquer le depot Git et `Jenkinsfile`.
4. Lancer le job et noter le numero du build.
5. Remplacer le tag `:1` des trois Deployments par ce numero, puis pousser le commit.

## 5. Argo CD

Une fois Argo CD installe dans son namespace :

```bash
kubectl apply -f argocd/application.yaml
```

L'application doit devenir `Synced` et `Healthy`. Les options `prune` et `selfHeal` sont activees dans le fichier.

## 6. Secrets en production

Le base64 n'est pas du chiffrement. Une personne qui lit le depot peut retrouver les mots de passe. En production, on peut utiliser Sealed Secrets, External Secrets Operator ou un gestionnaire comme HashiCorp Vault. Il faut aussi changer regulierement les secrets et limiter les droits d'acces.
