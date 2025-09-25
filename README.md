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

Faire les résolutions DNS :
* vip1 in A 195.15.196.89
* les domaines myapp-bluegreen.ingcloud.site, etc in CNAME vip1

Pour voir les applications :

```
# kubectl get applications -A
NAMESPACE   NAME                      SYNC STATUS   HEALTH STATUS
argocd      argo-rollouts             Synced        Healthy
argocd      ingress-nginx             Synced        Healthy
argocd      myapp-bluegreen-prod      Synced        Healthy
argocd      myapp-bluegreen-staging   Synced        Healthy
argocd      root-apps                 Synced        Healthy
```

Ou :

```
# argocd app list
```

Pour voir les rollouts :

```
# kubectl argo rollouts list rollouts -A
NAMESPACE                NAME             STRATEGY   STATUS        STEP  SET-WEIGHT  READY  DESIRED  UP-TO-DATE  AVAILABLE
myapp-blue-green-prod    myapp-bluegreen  BlueGreen  Healthy       -     -           6/6    6        6           6        
myapp-bluegreen-staging  myapp-bluegreen  BlueGreen  Healthy       -     -           6/6    6        6           6   
...
```

### Tester le blue-green en staging

Les 2 sites (active et preview) sont de la même couleur :
* http://staging-myapp-bluegreen.ingcloud.site/
* http://staging-myapp-bluegreen-preview.ingcloud.site/

On modifie l'image myapp-bluegreen-staging dans `myapps/myapp-bluegreen/overlays/staging/kustomization.yaml` (`images.newTag`) + COMMIT + PUSH

On sync l'appli en staging (ou via la GUI ou on attend 3min) :

```
# argocd app sync myapp-bluegreen-staging
```

L'application de staging passe en `suspended/paused`

```
# kubectl argo rollouts list rollouts -n 
  
```

L'app en preview est d'une autre couleur : http://staging-myapp-bluegreen-preview.ingcloud.site/



