# app-rollouts-demo

## Installation 

### Argocd


## Argo CLI

## Rollouts CLI

## Lancement

```
# kubectl apply -f apps/root-apps.yaml
```

### Tests

```
# k -n ingress-nginx get svc
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.233.63.232   195.15.196.89   80:30643/TCP,443:30856/TCP   88s
ingress-nginx-controller-admission   ClusterIP      10.233.53.197   <none>          443/TCP                      88s

# k get ingress -A
NAMESPACE                 NAME              CLASS   HOSTS                                           ADDRESS         PORTS   AGE
myapp-blue-green-prod     ingress-active    nginx   myapp-bluegreen.ingcloud.site                   195.15.196.89   80      2m30s
myapp-blue-green-prod     ingress-preview   nginx   myapp-bluegreen-preview.ingcloud.site           195.15.196.89   80      2m30s
myapp-bluegreen-staging   ingress-active    nginx   staging-myapp-bluegreen.ingcloud.site           195.15.196.89   80      2m30s
myapp-bluegreen-staging   ingress-preview   nginx   staging-myapp-bluegreen-preview.ingcloud.site   195.15.196.89   80      2m30s
```

* Faire les résolutions DNS

Pour voir les rollouts :

```
# kubectl argo rollouts list rollouts -A
 
```

### Tester le blue green en staging

On modifie l'image myapp-bluegreen-staging dans `myapps/myapp-bluegreen/overlays/staging/kustomization.yaml` + COMMIT + PUSH

On sync l'appli (ou on attend 3min)

```
# argocd app sync myapp-bluegreen-staging
```

L'application de staging passe en `suspended/paused`

```
# kubectl argo rollouts list rollouts -n 
  
```

