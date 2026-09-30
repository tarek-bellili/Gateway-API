# Gateway API / NGINX Gateway Fabric (NGF)

Ce composant installe les CRDs Gateway API (channel standard), le control plane NGINX Gateway Fabric (NGF), les certificats internes nécessaires à sa communication interne, et une `Gateway` partagée (`platform-gateway`) exposant Harbor, ArgoCD et Grafana derrière une IP unique.

## Architecture

Cette plateforme repose sur trois rôles distincts :

- **VM Mirror** — héberge le registre Docker (images) et le dépôt Helm (charts `.tgz`). C'est une source, rien ne s'y exécute.
- **VM PIC** — lance les playbooks Ansible (`ansible-playbook`), pilote le cluster via `kubectl`/`helm`. Contient les projets `bundle/` + `deploy-<composant>/`.
- **Cluster K8s** — exécute les workloads, doit pouvoir joindre le Mirror pour tirer images et charts.

**Point important** : la grande majorité des erreurs `ImagePullBackOff` ou `connection refused` rencontrées sur ce composant ne viennent pas de Gateway API ou de NGF eux-mêmes, mais d'un problème de connectivité réseau entre le cluster et la VM Mirror. Avant de creuser un composant en échec, toujours vérifier que le nœud concerné peut joindre `192.168.1.20:5000` (registre) et `192.168.1.20:8080` (dépôt Helm).

Ce composant introduit une **Gateway partagée unique**, `platform-gateway`, dans un namespace dédié `gateway-system`, avec un Listener HTTPS distinct par application (Harbor, ArgoCD, Grafana). Cette architecture remplace un modèle antérieur à une Gateway par composant, abandonné pour deux raisons :

1. Contrainte réseau — une seule IP MetalLB disponible pour l'ensemble des applications de la plateforme.
2. Une Gateway par composant impliquait davantage de surface à maintenir sans bénéfice d'isolation réel dans ce contexte.

Chaque application (Harbor, ArgoCD, Grafana) crée son propre `HTTPRoute`, dans son propre namespace, avec `parentRefs` pointant vers `platform-gateway`/`gateway-system` — géré dans le playbook de déploiement de chaque application respective, pas dans ce composant.

---

## 1️⃣ Déploiement

Depuis la VM PIC :

```bash
cd deploy-gatewayapi
ansible-playbook playbook-crds.yml            # installe les CRDs Gateway API
ansible-playbook playbook-certificates.yml    # crée les certificats internes NGF (server-tls / agent-tls)
ansible-playbook playbook.yml                 # déploie le control plane NGF (chart Helm)
ansible-playbook playbook-gateway-system.yml  # crée le namespace gateway-system, les Certificates applicatifs et platform-gateway
```

**Ordre strict à respecter** — chaque playbook dépend du précédent :

```
playbook-crds.yml
        ↓ (les CRDs Gateway API doivent exister)
playbook-certificates.yml
        ↓ (server-tls / agent-tls doivent exister avant le démarrage du control plane)
playbook.yml
        ↓ (le GatewayClass "nginx" doit être Accepted)
playbook-gateway-system.yml
```

> **Remarque** : en cas d'erreur `Invalid kube-config file`, ajouter `become: true` au play concerné — le kubeconfig par défaut n'est lisible qu'en `root` sur certains nœuds.

---

## 2️⃣ Vérifications

**CRDs installées** (8 au total, channel standard v1.5.1) :

```bash
kubectl get crds | grep gateway.networking.k8s.io
```

Attendu : `gatewayclasses`, `gateways`, `httproutes`, `referencegrants`, `grpcroutes`, `tlsroutes`, `backendtlspolicies`, `listenersets`.

**Pods du control plane NGF** :

```bash
kubectl get pods -n nginx-gateway
```

**GatewayClass acceptée** :

```bash
kubectl get gatewayclass
```

Attendu : `nginx` avec `ACCEPTED: True`.

**Gateway et data plane** :

```bash
kubectl get gateway -n gateway-system
kubectl get pods,svc -n gateway-system
```

Attendu : `platform-gateway` avec `PROGRAMMED: True` et une adresse IP assignée par MetalLB (`primary-pool`). Le `Service` correspondant doit être de type `LoadBalancer` avec cette même `EXTERNAL-IP`.

**HTTPRoutes des applications** (créés par les playbooks de chaque application, pas par ce composant) :

```bash
kubectl get httproute -A
```

Chaque route doit afficher `Accepted: True` sur sa condition de parent.

---

## 3️⃣ Test fonctionnel

Test end-to-end sur un hostname exposé, vérification du certificat TLS servi :

```bash
curl -v --resolve harbor.dev.local:443:<IP_PLATFORM_GATEWAY> https://harbor.dev.local
```

Le `-v` permet d'inspecter la chaîne de certificats retournée : elle doit correspondre au `Certificate` créé pour Harbor (`harbor-tls`), signé par `internal-ca-issuer`. Répéter pour `argocd.dev.local` et `grafana.dev.local` en changeant uniquement le hostname — la même IP sert les trois, le bon certificat est sélectionné via SNI.

---

## Contenu des playbooks

**`playbook-crds.yml`**

Rôle unique : applique les 8 CRDs Gateway API (channel standard, v1.5.1) en *server-side apply*, gérées hors du cycle de vie Helm (jamais recréées par NGF, jamais supprimées par un `helm uninstall` accidentel). Les CRDs sont lues depuis un manifeste brut récupéré sur GitHub — ce composant n'a pas de chart Helm dédié aux CRDs, contrairement à cert-manager.

**`playbook-certificates.yml`**

- *Create server-tls Certificate* : crée le `Certificate` `server-tls` (communication interne côté control plane NGF), signé par `internal-ca-issuer`
- *Create agent-tls Certificate* : crée le `Certificate` `agent-tls` (communication interne côté agent NGINX du data plane), signé par le même issuer
- *Wait for secrets* : attend la génération effective des deux `Secret` correspondants avant de continuer — le chart NGF ne doit jamais démarrer sans eux, puisque `certGenerator.enable: false` désactive la génération automatique par défaut du chart

**`playbook.yml`**

- *Create namespace* : crée `nginx-gateway`
- *Add Helm repository* : ajoute le dépôt Helm pointant vers la VM Mirror
- *Render values* : génère `values-nginx-gateway-fabric.yaml` à partir du template, avec le registre Mirror en repository pour les deux images (control plane et data plane) et `certGenerator.enable: false`
- *Deploy via Helm* : installe le chart `nginx-gateway-fabric` (control plane NGF)
- *Verify pods Running* : attend que le pod du control plane soit opérationnel

**`playbook-gateway-system.yml`**

- *Create gateway-system namespace* : crée le namespace transverse `gateway-system`
- *Create Certificate for Harbor / ArgoCD / Grafana* : crée les trois `Certificate` applicatifs, tous signés par `internal-ca-issuer`, centralisés dans `gateway-system` plutôt que dans le namespace de chaque application
- *Wait for secrets* : attend la génération des trois `Secret` TLS correspondants
- *Create platform-gateway* : crée la `Gateway` partagée, avec un Listener HTTPS par application (référençant chacun son propre `Secret`) et l'annotation MetalLB fixant le pool d'IP (`primary-pool`)
- *Verify Gateway Programmed* : attend que `platform-gateway` atteigne l'état `Programmed: True`, confirmant que MetalLB a bien assigné une IP et que le data plane a démarré

---

## Variables clés (`group_vars/all.yml`)

```yaml
# --- Gateway API ---
gateway_api_version: "1.5.1"
gateway_api_release_channel: standard
gateway_api_crds_manifest_path: "{{ playbook_dir }}/../bundle/crds/gateway-api.crds.yaml"

# --- NGF : Helm release ---
ngf_namespace: nginx-gateway
ngf_release_name: ngf
ngf_chart_version: "2.6.7"

# --- NGF : certificats internes control plane <-> agent (via cert-manager) ---
ngf_server_tls_secret_name: server-tls
ngf_agent_tls_secret_name: agent-tls
ngf_server_tls_dns_name: "ngf-nginx-gateway-fabric.nginx-gateway.svc"
ngf_agent_tls_dns_name: "*.cluster.local"
ngf_ca_issuer_name: internal-ca-issuer

# --- NGF : chart Helm (dépôt interne) ---
ngf_chart_name: nginx-gateway-fabric
ngf_repo_name: gatewayapi-repo
ngf_repo_url: "http://192.168.1.20:8080"

# --- NGF : registre d'images privé ---
image_registry: "192.168.1.20:5000/gatewayapi"
image_tag: "2.6.7"

# --- NGF : dimensionnement ---
ngf_controlplane_replicas: 1
ngf_dataplane_replicas: 1

# --- NGF : GatewayClass ---
ngf_gatewayclass_name: nginx

# --- Gateway System (namespace transverse, Gateway mutualisée) ---
gateway_system_namespace: gateway-system
platform_gateway_name: platform-gateway
platform_gateway_metallb_pool: "primary-pool"

# --- Listeners du platform-gateway (un par application) ---
harbor_hostname: "harbor.dev.local"
argocd_hostname: "argocd.dev.local"
grafana_hostname: "grafana.dev.local"

# --- Certificats applicatifs centralisés dans gateway-system ---
harbor_tls_secret_name: harbor-tls
argocd_tls_secret_name: argocd-tls
grafana_tls_secret_name: grafana-tls
```

> **Note d'architecture** — chaque `Gateway` provisionne dynamiquement son propre data plane NGINX (Deployment + Service LoadBalancer), dans le même namespace que le `Gateway` lui-même. `nginx.replicas` dans le chart NGF est un réglage **global**, appliqué par défaut à tous les Gateway existants ; un dimensionnement différencié par Gateway nécessite un `NginxProxy` dédié, référencé via `infrastructure.parametersRef`.

---

## Pratique : ajouter un nouveau Listener / une nouvelle application sur `platform-gateway`

1. **Créer le `Certificate`** pour la nouvelle application, dans `gateway-system`, signé par `internal-ca-issuer` :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: <app>
  namespace: gateway-system
spec:
  secretName: <app>-tls
  dnsNames:
    - <app>.dev.local
  issuerRef:
    name: internal-ca-issuer
    kind: ClusterIssuer
```

2. **Ajouter un Listener** à `platform-gateway` (edit ou re-apply du manifeste), avec un nom unique :

```yaml
- name: <app>
  port: 443
  protocol: HTTPS
  hostname: "<app>.dev.local"
  tls:
    mode: Terminate
    certificateRefs:
      - kind: Secret
        name: <app>-tls
  allowedRoutes:
    namespaces:
      from: All
```

3. **Créer le `HTTPRoute`** dans le namespace de la nouvelle application, avec `sectionName` pointant explicitement vers le nom du Listener créé à l'étape 2 (recommandé dès qu'il y a plus d'un Listener sur la Gateway) :

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <app>-route
  namespace: <namespace-app>
spec:
  parentRefs:
    - name: platform-gateway
      namespace: gateway-system
      sectionName: <app>
  hostnames:
    - "<app>.dev.local"
  rules:
    - backendRefs:
        - name: <service-backend>
          port: <port>
```

4. **Vérifier** : `kubectl describe httproute <app>-route -n <namespace-app>` doit afficher `Accepted: True` sur la condition de parent.
