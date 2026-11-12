# TP Kubernetes - Bijouterie

Application microservices composee d'un frontend Nginx, d'une API Node.js et d'une base MySQL.

- Depot GitHub : https://github.com/digitalartist221/devops-bijouterie
- Images Docker : https://hub.docker.com/repositories/digitalartistsn

## Test avec Docker Compose

```bash
docker compose up --build -d
docker compose ps
curl http://localhost:8080/api/config
```

Pour arreter l'application :

```bash
docker compose down
```

## Test Kubernetes avec kind

```bash
kind create cluster --name bijouterie --config kind-config.yaml

docker build -t digitalartistsn/bijouterie-frontend:1 frontend/
docker build -t digitalartistsn/bijouterie-backend:1 backend/
docker build -t digitalartistsn/bijouterie-mysql:1 database/

kind load docker-image digitalartistsn/bijouterie-frontend:1 --name bijouterie
kind load docker-image digitalartistsn/bijouterie-backend:1 --name bijouterie
kind load docker-image digitalartistsn/bijouterie-mysql:1 --name bijouterie

kubectl apply -f k8s/
kubectl get all -n bijouterie
kubectl get configmap,secret -n bijouterie
kubectl get endpoints -n bijouterie
curl http://localhost:30080/api/config
```

## Jenkins

1. Creer un Access Token sur Docker Hub.
2. Dans Jenkins, ouvrir **Manage Jenkins > Credentials**.
3. Ajouter un credential **Username with password** :
   - Username : `digitalartistsn`
   - Password : le token Docker Hub
   - ID : `dockerhub-credentials`
4. Creer un job **Pipeline** avec **Pipeline script from SCM**.
5. Choisir Git et utiliser `https://github.com/digitalartist221/devops-bijouterie.git`.
6. Indiquer `Jenkinsfile` comme chemin du script, puis lancer le build.

Le pipeline construit et publie les trois images avec le numero du build et le tag `latest`. Les manifests Kubernetes utilisent un numero de build explicite.

## Installer Argo CD

Verifier d'abord que `kubectl` utilise le bon cluster :

```bash
kubectl config current-context
```

Installer Argo CD dans son namespace :

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Available deployment --all -n argocd --timeout=300s
kubectl get pods -n argocd
```

## Ouvrir Argo CD

Dans un terminal, lancer :

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Ouvrir ensuite https://localhost:8080. Le certificat est auto-signe, le navigateur peut afficher un avertissement.

Le nom d'utilisateur est `admin`. Pour obtenir le mot de passe initial :

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo
```

## Configurer l'application Argo CD

Le depot GitHub est public. Il n'est donc pas necessaire d'ajouter un token GitHub dans Argo CD.

```bash
kubectl apply -f argocd/application.yaml
kubectl get applications -n argocd
kubectl describe application bijouterie -n argocd
```

Dans l'interface Argo CD, l'application `bijouterie` doit passer a **Synced** et **Healthy**. Elle surveille la branche `main`, le dossier `k8s` et deploie dans le namespace `bijouterie`.

La synchronisation automatique est activee avec :

- `prune: true` pour supprimer les ressources retirees de Git ;
- `selfHeal: true` pour corriger les changements faits manuellement ;
- `CreateNamespace=true` pour creer le namespace de destination.

Une fois Argo CD en place, les changements de l'application doivent passer par Git. Il ne faut plus utiliser `kubectl apply -f k8s/` manuellement.

## Secrets

Les valeurs base64 des Secrets ne sont pas chiffrees. En production, il faut utiliser une solution comme Sealed Secrets, External Secrets Operator ou HashiCorp Vault.
