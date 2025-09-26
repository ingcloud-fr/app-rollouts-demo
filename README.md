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

## External DNS (openstack designate)

### Creation des zones

* Installer le client openstack dns designate

```
$ sudo apt install python3-designate
```

* Note : on a aussi `python3-openstackclient` et `python3-octavia` d'installés

* Créer une zone k8s.ingcloud.site avec la GUI ou en ligne de commande (ne pas oublier le . à la fin du domaine):

```
$ openstack zone create --email hostmaster@ingcloud.site k8s.ingcloud.site.
+----------------+--------------------------------------+
| Field          | Value                                |
+----------------+--------------------------------------+
| action         | CREATE                               |
| attributes     |                                      |
| created_at     | 2025-09-26T16:42:10.000000           |
| description    | None                                 |
| email          | hostmaster@ingcloud.site             |
| id             | e87f68a9-1bf6-4edc-8fab-a5a64637a715 |
| masters        |                                      |
| name           | k8s.ingcloud.site.                  |
| pool_id        | 794ccc2c-d751-44fe-b57f-8894c9f5c842 |
| project_id     | da72895b87e24bbf82133eee25908b39     |
| serial         | 1758904930                           |
| status         | PENDING                              |
| transferred_at | None                                 |
| ttl            | 3600                                 |
| type           | PRIMARY                              |
| updated_at     | None                                 |
| version        | 1                                    |
+----------------+--------------------------------------+

```

On récupère le NS de la zone créée :

```
$ openstack recordset list k8s.ingcloud.site.
+-------------------------------+-------------------------------+------+---------------------------------+--------+--------+
| id                            | name                          | type | records                         | status | action |
+-------------------------------+-------------------------------+------+---------------------------------+--------+--------+
| 20fa89e3-7c6a-4267-b72a-      | k8s.ingcloud.site.            | NS   | ns1.pub1.infomaniak.cloud.      | ACTIVE | NONE   |
| 33bd02025e00                  |                               |      | ns2.pub1.infomaniak.cloud.      |        |        |
| b56d28c7-d80d-4d39-b4bf-      | k8s.ingcloud.site.            | SOA  | ns2.pub1.infomaniak.cloud.      | ACTIVE | NONE   |
+-------------------------------+-------------------------------+------+---------------------------------+--------+--------+
```

Sur le registar du domaine **ingcloud.site**, faire un nouveau enregistrement de type **NS** `k8s.ingcloud.site vers` `ns1.pub1.infomaniak.cloud.` et `ns2.pub1.infomaniak.cloud.`

### External-DNS

* Doc: https://kubernetes-sigs.github.io/external-dns/v0.15.1/docs/tutorials/designate/

On crée le ns external-dns 

```
# kubectl create ns external-dns
```

On crée le secret (cf openrc.sh pour les valeurs) dans le ns external-dns :

```
# kubectl -n external-dns create secret generic os-openrc  \ 
    --from-literal=OS_AUTH_URL='https://api.pub1.infomaniak.cloud/identity/v3' \
    --from-literal=OS_REGION_NAME='dc3-a' \
    --from-literal=OS_USERNAME='PCU-xxxxx' \
    --from-literal=OS_PASSWORD='xxxxxxx' \ 
    --from-literal=OS_PROJECT_NAME='PCP-xxxxx' \
    --from-literal=OS_PROJECT_ID='xxxxxx' \
    --from-literal=OS_USER_DOMAIN_NAME='default' \
    --from-literal=OS_PROJECT_DOMAIN_NAME='default' \
    --dry-run=client -o yaml | kubectl apply -f -
```

ou (mais ne pas commiter) :

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: os-openrc
  namespace: external-dns
type: Opaque
stringData:
  OS_AUTH_URL: "https://api.pub1.infomaniak.cloud/identity/v3"
  OS_REGION_NAME: "dc3-a"
  OS_USERNAME: "PCU-xxxx"
  OS_PASSWORD: "<ton_mot_de_passe>"
  OS_PROJECT_NAME: "PCP-xxxx"
  OS_PROJECT_ID: "xxxxxxx"
  OS_USER_DOMAIN_NAME: "default"
  OS_PROJECT_DOMAIN_NAME: "default"`
```

Le fichier `external-dns.yaml`:

```yaml
# external-dns.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  namespace: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services","endpoints","pods"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions","networking.k8s.io"]
  resources: ["ingresses"] 
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["watch","list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: external-dns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: external-dns
spec:
  selector:
    matchLabels:
      app: external-dns
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: registry.k8s.io/external-dns/external-dns:v0.16.1 # dernière version avec provider designate (v0.17+ passer avec provider webhook+webhook Designate)
        args:
        - --source=ingress # service is also possible
        - --domain-filter=k8s.ingcloud.site # (optional) limit to only example.com domains; change to match the zone created above.
        - --provider=designate
        - --registry=txt # Active le “registre” TXT : ExternalDNS crée/maintient un TXT par nom géré pour enregistrer l’ownership
        - --txt-prefix=_extdns. # les TXT seront créés sur _extdns.xxxx.xx
        - --policy=sync # crée, met à jour et supprime pour coller à l’état K8s (autre "upsert-only")
        - --txt-owner-id=c1-k8s # C'est le cluster k8s c1 qui est owner (utile si plusieurs clusters)
        env:
          - name: OS_AUTH_URL
            valueFrom:
             secretKeyRef:
               name: os-openrc
               key: OS_AUTH_URL
          - name: OS_REGION_NAME
            valueFrom:
              secretKeyRef:
                name: os-openrc
                key: OS_REGION_NAME
          - name: OS_USERNAME
            valueFrom:
              secretKeyRef:
                name: os-openrc
                key: OS_USERNAME
          - name: OS_PASSWORD
            valueFrom:
              secretKeyRef:
                name: os-openrc
                key: OS_PASSWORD
          - name: OS_PROJECT_NAME
            valueFrom:
              secretKeyRef:
                name: os-openrc
                key: OS_PROJECT_NAME
          - name: OS_PROJECT_ID
            valueFrom:
              secretKeyRef:
                name: os-openrc
                key: OS_PROJECT_ID
          - name: OS_USER_DOMAIN_NAME
            valueFrom:
              secretKeyRef:
                name: os-openrc
                key: OS_USER_DOMAIN_NAME
          - name: OS_PROJECT_DOMAIN_NAME
            valueFrom:
              secretKeyRef:
                name: os-openrc
                key: OS_PROJECT_DOMAIN_NAME
```

```
# k apply -f external-dns.yaml 
namespace/external-dns configured
serviceaccount/external-dns created
clusterrole.rbac.authorization.k8s.io/external-dns created
clusterrolebinding.rbac.authorization.k8s.io/external-dns-viewer created
deployment.apps/external-dns created

```

```
# kubectl logs deploy/external-dns -f
...
time="2025-09-26T17:09:20Z" level=info msg="Instantiating new Kubernetes client"
time="2025-09-26T17:09:20Z" level=info msg="Using inCluster-config based on serviceaccount-token"
time="2025-09-26T17:09:20Z" level=info msg="Created Kubernetes client https://10.233.0.1:443"
time="2025-09-26T17:09:20Z" level=info msg="Using OpenStack Keystone at https://api.pub1.infomaniak.cloud/identity/v3/"
time="2025-09-26T17:09:21Z" level=info msg="Found OpenStack Designate service at https://api.pub1.infomaniak.cloud/dns/"
time="2025-09-26T17:09:21Z" level=info msg="All records are already up to date"
```

## Lancement de l'app-root

```
# kubectl apply -f apps/root-apps.yaml
```

### Tests Ingress

```
# k get ingressclasses
NAME            CONTROLLER                      PARAMETERS   AGE
nginx-prod      k8s.io/ingress-nginx            <none>       3m38s
nginx-staging   k8s.io/ingress-nginx            <none>       3m38s


# k -n ingress-nginx-prod get svc
NAME                                      TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx-prod-controller             LoadBalancer   10.233.32.92   195.15.197.215   80:30717/TCP,443:30783/TCP   4m28s
ingress-nginx-prod-controller-admission   ClusterIP      10.233.44.92   <none>           443/TCP                      4m28s

# k -n ingress-nginx-staging get svc
NAME                                         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-staging-controller             LoadBalancer   10.233.41.54    195.15.199.45   80:32017/TCP,443:31436/TCP   4m51s
ingress-nginx-staging-controller-admission   ClusterIP      10.233.37.222   <none>          443/TCP                      4m51s



# k get ingress -A
NAMESPACE                 NAME                                 CLASS           HOSTS                                               ADDRESS          PORTS   AGE
myapp-blue-green-prod     ingress-active                       nginx-prod      myapp-bluegreen.k8s.ingcloud.site                   195.15.197.215   80      5m42s
myapp-blue-green-prod     ingress-preview                      nginx-prod      myapp-bluegreen-preview.k8s.ingcloud.site           195.15.197.215   80      5m42s
myapp-bluegreen-staging   ingress-active                       nginx-staging   staging-myapp-bluegreen.k8s.ingcloud.site           195.15.199.45    80      5m43s
myapp-bluegreen-staging   ingress-preview                      nginx-staging   staging-myapp-bluegreen-preview.k8s.ingcloud.site   195.15.199.45    80      5m43s
myapp-canary-prod         ingress-stable                       nginx-prod      myapp-canary.k8s.ngcloud.site                       195.15.197.215   80      5m42s
myapp-canary-prod         myapp-canary-ingress-stable-canary   nginx-prod      myapp-canary.k8s.ngcloud.site                       195.15.197.215   80      3m10s
myapp-canary-staging      ingress-stable                       nginx-staging   staging-myapp-canary.k8s.ingcloud.site              195.15.199.45    80      5m42s
myapp-canary-staging      myapp-canary-ingress-stable-canary   nginx-staging   staging-myapp-canary.k8s.ingcloud.site              195.15.199.45    80      3m12s

```

Vérifier les logs d'external-dns :

```
# kubectl logs deploy/external-dns -f
...
time="2025-09-26T16:23:28Z" level=info msg="Creating records: staging-myapp-bluegreen.k8s.ingcloud.site./A: 195.15.199.45"
time="2025-09-26T16:23:28Z" level=info msg="Creating records: _extdns.staging-myapp-bluegreen.k8s.ingcloud.site./TXT: \"heritage=external-dns,external-dns/owner=c1-k8s,external-dns/resource=ingress/myapp-bluegreen-staging/ingress-active\""
time="2025-09-26T16:23:28Z" level=info msg="Creating records: _extdns.a-staging-myapp-bluegreen.k8s.ingcloud.site./TXT: \"heritage=external-dns,external-dns/owner=c1-k8s,external-dns/resource=ingress/myapp-bluegreen-staging/ingress-active\""
time="2025-09-26T16:23:28Z" level=info msg="Creating records: _extdns.staging-myapp-bluegreen-preview.k8s.ingcloud.site./TXT: \"heritage=external-dns,external-dns/owner=c1-k8s,external-dns/resource=ingress/myapp-bluegreen-staging/ingress-preview\""
time="2025-09-26T16:23:28Z" level=info msg="Creating records: staging-myapp-bluegreen-preview.k8s.ingcloud.site./A: 195.15.199.45"
time="2025-09-26T16:23:29Z" level=info msg="Creating records: staging-myapp-canary.k8s.ingcloud.site./A: 195.15.199.45"
time="2025-09-26T16:23:29Z" level=info msg="Creating records: _extdns.staging-myapp-canary.k8s.ingcloud.site./TXT: \"heritage=external-dns,external-dns/owner=c1-k8s,external-dns/resource=ingress/myapp-canary-staging/ingress-stable\""
time="2025-09-26T16:24:29Z" level=info msg="Creating records: myapp-bluegreen.k8s.ingcloud.site./A: 195.15.197.215"
time="2025-09-26T16:24:29Z" level=info msg="Creating records: myapp-bluegreen-preview.k8s.ingcloud.site./A: 195.15.197.215"
time="2025-09-26T16:25:30Z" level=info msg="All records are already up to date"
...
```

Dans l'interface Horizon > DNS , on peut voir la zone k8s.ingloud.site et ses enregistrements

* Note il n'y a que 5 enregistrements car certains sont en double.


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
myapp-canary-prod        myapp-canary     Canary     Healthy       7/7   100         6/6    6        6           6        
myapp-canary-staging     myapp-canary     Canary     Healthy       7/7   100         6/6    6        6           6 
...
```

### Tester le blue-green en staging

Les 2 sites (active et preview) sont de la même couleur :
* http://staging-myapp-bluegreen.k8s.ingcloud.site/
* http://staging-myapp-bluegreen-preview.k8s.ingcloud.site/

On modifie l'image myapp-bluegreen-staging dans `myapps/myapp-bluegreen/overlays/staging/kustomization.yaml` (`images.newTag`) + COMMIT + PUSH

On sync l'appli en staging (ou via la GUI ou on attend 3min) :

```
# argocd app sync myapp-bluegreen-staging
```

* Le controlleur Argo Rollout détecte le changement d'image
* L'application de staging passe en `suspended/paused`
* L'app en preview est d'une autre couleur : http://staging-myapp-bluegreen-preview.k8s.ingcloud.site/


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

Idem staging avec urls :

* http://myapp-bluegreen.k8s.ingcloud.site/
* http://staging-myapp-bluegreen.ingcloud.site/
* Modif de `myapps/myapp-bluegreen/overlays/prod/kustomization.yaml` (`image.newTag`)
* `# argocd app sync myapp-bluegreen-prod`
* Voir le rollout : `# kubectl argo rollouts get rollout myapp-bluegreen -n myapp-bluegreen-prod`
* `# kubectl argo rollouts promote myapp-bluegreen -n myapp-bluegreen-prod`





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

