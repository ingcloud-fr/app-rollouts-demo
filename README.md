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

Le controlleur Argo Rollout détecte le changement d'image
L'application de staging passe en `suspended/paused`
L'app en preview est d'une autre couleur : http://staging-myapp-bluegreen-preview.ingcloud.site/


On a une stratégie rollout manuelle (cf `myapps/myapp-bluegreen/base/rollout.yaml`) avec `autoPromotionEnabled: false` et 2 replicas pour la preview :

```
# kubectl argo rollouts get rollout myapp-bluegreen -n myapp-bluegreen-staging
Name:            myapp-bluegreen
Namespace:       myapp-bluegreen-staging
Status:          ॥ Paused
Message:         BlueGreenPause
Strategy:        BlueGreen
Images:          argoproj/rollouts-demo:red (stable, active)
                 argoproj/rollouts-demo:yellow (preview)
Replicas:
  Desired:       6
  Current:       8
  Updated:       2
  Ready:         6
  Available:     6

NAME                                         KIND        STATUS     AGE    INFO
⟳ myapp-bluegreen                            Rollout     ॥ Paused   38m    
├──# revision:2                                                            
│  └──⧉ myapp-bluegreen-7d7db49cf4           ReplicaSet  ✔ Healthy  4m55s  preview
│     ├──□ myapp-bluegreen-7d7db49cf4-6fds4  Pod         ✔ Running  4m55s  ready:1/1
│     └──□ myapp-bluegreen-7d7db49cf4-8rxwx  Pod         ✔ Running  4m55s  ready:1/1
└──# revision:1                                                            
   └──⧉ myapp-bluegreen-58cbc6f97            ReplicaSet  ✔ Healthy  38m    stable,active
      ├──□ myapp-bluegreen-58cbc6f97-26rlc   Pod         ✔ Running  37m    ready:1/1
      ├──□ myapp-bluegreen-58cbc6f97-95f4p   Pod         ✔ Running  37m    ready:1/1
      ├──□ myapp-bluegreen-58cbc6f97-9jr7r   Pod         ✔ Running  37m    ready:1/1
      ├──□ myapp-bluegreen-58cbc6f97-cc64g   Pod         ✔ Running  37m    ready:1/1
      ├──□ myapp-bluegreen-58cbc6f97-hnf9x   Pod         ✔ Running  37m    ready:1/1
      └──□ myapp-bluegreen-58cbc6f97-vv868   Pod         ✔ Running  37m    ready:1/1
```

#### Promouvoir le preview :


```
# kubectl argo rollouts promote myapp-bluegreen -n myapp-bluegreen-staging
rollout 'myapp-bluegreen' promoted
```



Au bout de 30 sec :

```
# kubectl argo rollouts get rollout myapp-bluegreen -n myapp-bluegreen-staging
Name:            myapp-bluegreen
Namespace:       myapp-bluegreen-staging
Status:          ✔ Healthy
Strategy:        BlueGreen
Images:          argoproj/rollouts-demo:yellow (stable, active)
Replicas:
  Desired:       6
  Current:       6
  Updated:       6
  Ready:         6
  Available:     6

NAME                                         KIND        STATUS        AGE  INFO
⟳ myapp-bluegreen                            Rollout     ✔ Healthy     44m  
├──# revision:2                                                             
│  └──⧉ myapp-bluegreen-7d7db49cf4           ReplicaSet  ✔ Healthy     10m  stable,active
│     ├──□ myapp-bluegreen-7d7db49cf4-6fds4  Pod         ✔ Running     10m  ready:1/1
│     ├──□ myapp-bluegreen-7d7db49cf4-8rxwx  Pod         ✔ Running     10m  ready:1/1
│     ├──□ myapp-bluegreen-7d7db49cf4-9brzv  Pod         ✔ Running     44s  ready:1/1
│     ├──□ myapp-bluegreen-7d7db49cf4-hhvdc  Pod         ✔ Running     44s  ready:1/1
│     ├──□ myapp-bluegreen-7d7db49cf4-p2l4n  Pod         ✔ Running     44s  ready:1/1
│     └──□ myapp-bluegreen-7d7db49cf4-rm5zs  Pod         ✔ Running     44s  ready:1/1
└──# revision:1                                                             
   └──⧉ myapp-bluegreen-58cbc6f97            ReplicaSet  • ScaledDown  43m 
```

Le service active et preview sont de la même couleur maintenant.

#### Revenir en arrière

Pour ne pas promouvoir la preview :

```
kubectl argo rollouts abort <ROLLOUT_NAME> -n <ns>

```
Si on s'apercoit qu'il y a une erreur (après la promotion) pour revenir en arrière :

```
kubectl argo rollouts undo <ROLLOUT_NAME> -n <ns>
```

Attention: on se retrouve en drift avec la verison Git et l’auto-sync/selfHeal, Argo CD va réappliquer ce qu’il y a dans Git et “annuler l'annulation”

Il vaut mieux modifier l'image dans Git + COMMIT + PUSH + APP SYNC

* Note On peut faire du Writeback git avec image-updater.

#### Tester le blue-green en prod

Idem staging


### Tester le canary en staging

```
# k get ingress -n myapp-canary-staging
NAME                                 CLASS   HOSTS                                ADDRESS         PORTS   AGE
ingress-stable                       nginx   staging-myapp-canary.ingcloud.site   195.15.196.89   80      25m
myapp-canary-ingress-stable-canary   nginx   staging-myapp-canary.ingcloud.site   195.15.196.89   80      25m
```

Notes : 
* Il est normal d’avoir 2 Ingress avec le même host en stratégie canary avec NGINX avec même règle/path. Le pattern standard est : un Ingress “stable” + un Ingress “canary” (même host), et NGINX route une fraction du trafic vers le canary en fonction des annotations.
* L'ingress `myapp-canary-ingress-stable-canary` est automatiquement crée par argocd rollouts

On peut voir les annotations :

```
# kubectl -n myapp-canary-staging describe ing ingress-stable
Name:             ingress-stable
...
Rules:
  Host                                Path  Backends
  ----                                ----  --------
  staging-myapp-canary.ingcloud.site  
                                      /   svc-stable:80 (10.233.66.19:8080,10.233.66.221:8080,10.233.66.27:8080 + 3 more...)
Annotations:                          argocd.argoproj.io/tracking-id: myapp-canary-staging:networking.k8s.io/Ingress:myapp-canary-staging/ingress-stable
...
```

```
# kubectl -n myapp-canary-staging describe ing myapp-canary-ingress-stable-canary
Name:             myapp-canary-ingress-stable-canary
...
Rules:
  Host                                Path  Backends
  ----                                ----  --------
  staging-myapp-canary.ingcloud.site  
                                      /   svc-canary:80 (10.233.66.19:8080,10.233.66.221:8080,10.233.66.27:8080 + 3 more...)
Annotations:                          nginx.ingress.kubernetes.io/canary: true
                                      nginx.ingress.kubernetes.io/canary-weight: 0
...
```

On fait l'entrée DNS : staging-myapp-canary.ingcloud.site CNAME de vip1

Si on va sur http://staging-myapp-canary.ingcloud.site est est à 100% d'une couleur

On change l'image dans `myapps/myapp-canary/overlays/staging/kustomization.yaml` (dans `image.newTag`)



