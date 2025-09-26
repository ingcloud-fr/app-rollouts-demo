# app-rollouts-demo

* 2 ingress controlleur (prod et staging) dans 2 namespaces

## Installation 

```
# git clone xxxx
cd xxxx
```

### Argocd

On installe argocd via l'overlay (création du namespace, installation des CRDs, etc) avec patch service LB ;

```
# kubectl apply -k bootstrap/argocd/overlays/standard/
```

* Note: Si 3 workers ou plus, on peut installer la version HA (NOT READY)

```
# kubectl apply -k bootstrap/argocd/overlays/ha/
```

On récupère l'adresse IP :

```
# kubectl -n argocd get svc argocd-server
```

Faire l'entrée DNS.

Récupèrer le mot de passe admin initial :

```
# kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```


## Argo CLI

On installe Argo CLI (sur poste client / master)

* https://argo-cd.readthedocs.io/en/stable/cli_installation/

Sur le poste client :

```
$ VERSION=$(curl -L -s https://raw.githubusercontent.com/argoproj/argo-cd/stable/VERSION) 
$ curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/v$VERSION/argocd-linux-amd64 
$ sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd 
$ rm argocd-linux-amd64
```
Test :

```
$ argocd version --client
argocd: v3.1.5+cfeed49
  BuildDate: 2025-09-10T16:01:20Z
  GitCommit: cfeed4910542c359f18537a6668d4671abd3813b
  GitTreeState: clean
  GoVersion: go1.24.6
  Compiler: gc
  Platform: linux/amd64
```

Pour récupérer l'IP + mot de passe

```
# kubectl -n argocd get svc argocd-server
# kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

```
$ argocd login <IP> --username admin --password '<PASSWD>' --insecure
```

Note: `--insecure` car certificat autosigné sinon avertissement



## Rollouts CLI : plugin `kubectl-argo-rollouts`

* Pratique pour suivre les déploiements blue/green & canary.
* Pour la version : https://github.com/argoproj/argo-rollouts

```bash
# Linux amd64 — adapte l’OS/arch et <VERSION>
curl -LO https://github.com/argoproj/argo-rollouts/releases/download/<VERSION>/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

# Test
kubectl argo rollouts version
```

Lancer le **dashboard** via le plugin (sans Helm) :

```bash
kubectl argo rollouts dashboard -n argo-rollouts
```


## Lancement

```
# kubectl apply -f apps/root-apps.yaml
```

### Tests Ingess

```
# k get ingressclasses
NAME            CONTROLLER                      PARAMETERS   AGE
nginx-prod      k8s.io/ingress-nginx            <none>       3m38s
nginx-staging   k8s.io/ingress-nginx            <none>       3m38s


# k get svc -n ingress-nginx-prod 
NAME                                      TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx-prod-controller             LoadBalancer   10.233.59.32   195.15.196.151   80:31193/TCP,443:31451/TCP   5m24s
ingress-nginx-prod-controller-admission   ClusterIP      10.233.10.5    <none>           443/TCP                      5m24s

# k get svc -n ingress-nginx-staging 
NAME                                         TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx-staging-controller             LoadBalancer   10.233.9.103    195.15.197.215   80:30277/TCP,443:31486/TCP   5m33s
ingress-nginx-staging-controller-admission   ClusterIP      10.233.44.113   <none>           443/TCP                      5m33s


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

On peut voir :

```
# kubectl get app -A
NAMESPACE   NAME                      SYNC STATUS   HEALTH STATUS
...
argocd      myapp-canary-staging      Synced        Suspended
```

On peut voir :

```
# kubectl argo rollouts get rollout myapp-canary -n myapp-canary-staging
Name:            myapp-canary
Namespace:       myapp-canary-staging
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/7         <--- ICI 1ier étape sur 7 (setWeight et pauses)
  SetWeight:     10          <--- 1ier setWeight
  ActualWeight:  10
Images:          argoproj/rollouts-demo:blue (stable)
                 argoproj/rollouts-demo:green (canary)
Replicas:
  Desired:       6
  Current:       7
  Updated:       1
  Ready:         7
  Available:     7

NAME                                      KIND        STATUS     AGE    INFO
⟳ myapp-canary                            Rollout     ॥ Paused   42m    
├──# revision:2                                                         
│  └──⧉ myapp-canary-744ddfc74d           ReplicaSet  ✔ Healthy  2m16s  canary
│     └──□ myapp-canary-744ddfc74d-5bs2q  Pod         ✔ Running  2m15s  ready:1/1
└──# revision:1                                                         
   └──⧉ myapp-canary-6bb996f685           ReplicaSet  ✔ Healthy  42m    stable
      ├──□ myapp-canary-6bb996f685-25dw7  Pod         ✔ Running  42m    ready:1/1
      ├──□ myapp-canary-6bb996f685-4vgbm  Pod         ✔ Running  42m    ready:1/1
      ├──□ myapp-canary-6bb996f685-6ljtz  Pod         ✔ Running  42m    ready:1/1
      ├──□ myapp-canary-6bb996f685-cnfqd  Pod         ✔ Running  42m    ready:1/1
      ├──□ myapp-canary-6bb996f685-mslwx  Pod         ✔ Running  42m    ready:1/1
      └──□ myapp-canary-6bb996f685-npnhs  Pod         ✔ Running  42m    ready:1/1
```

L'annotation de l'ingress canary :

```
# kubectl -n myapp-canary-staging describe ing myapp-canary-ingress-stable-canary
...
Annotations:                          nginx.ingress.kubernetes.io/canary: true
                                      nginx.ingress.kubernetes.io/canary-weight: 10
```


Si on regarde http://staging-myapp-canary.ingcloud.site/, il y a 2 couleurs : beaucoup de la 1ier, un peu de l'autre.


On valide cette 1ier étape :

```
# kubectl argo rollouts promote myapp-canary -n myapp-canary-staging
```

Notes :
* Promouvoir directement à 100% : `kubectl argo rollouts promote myapp-canary -n myapp-canary-staging --full`
* Annuler le déploiement en cours (rester sur la stable): `kubectl argo rollouts abort myapp-canary -n myapp-canary-staging`
* Annuler si déjà promu (retour arrière): `kubectl argo rollouts undo myapp-canary -n myapp-canary-staging`

On peut suivre le déploiement (les autres étapes sont en auto) :

```
# kubectl argo rollouts get rollout myapp-canary -n myapp-canary-staging -w
```

Si on regarde http://staging-myapp-canary.ingcloud.site/, il y a 2 couleurs : de moins en moins de la 1ier, de plus en plus de l'autre. Jusqu'à atteindre 100%.

```
# kubectl argo rollouts get rollout myapp-canary -n myapp-canary-staging
Name:            myapp-canary
Namespace:       myapp-canary-staging
Status:          ✔ Healthy
Strategy:        Canary
  Step:          7/7
  SetWeight:     100
  ActualWeight:  100
Images:          argoproj/rollouts-demo:green (stable)
Replicas:
  Desired:       6
  Current:       6
  Updated:       6
  Ready:         6
  Available:     6

NAME                                      KIND        STATUS        AGE    INFO
⟳ myapp-canary                            Rollout     ✔ Healthy     56m    
├──# revision:2                                                            
│  └──⧉ myapp-canary-744ddfc74d           ReplicaSet  ✔ Healthy     15m    stable
│     ├──□ myapp-canary-744ddfc74d-5bs2q  Pod         ✔ Running     15m    ready:1/1
│     ├──□ myapp-canary-744ddfc74d-bjf8j  Pod         ✔ Running     4m57s  ready:1/1
│     ├──□ myapp-canary-744ddfc74d-59t4z  Pod         ✔ Running     3m54s  ready:1/1
│     ├──□ myapp-canary-744ddfc74d-j7nh6  Pod         ✔ Running     3m54s  ready:1/1
│     ├──□ myapp-canary-744ddfc74d-55gff  Pod         ✔ Running     113s   ready:1/1
│     └──□ myapp-canary-744ddfc74d-x7ztv  Pod         ✔ Running     113s   ready:1/1
└──# revision:1                                                            
   └──⧉ myapp-canary-6bb996f685           ReplicaSet  • ScaledDown  56m  
```  

