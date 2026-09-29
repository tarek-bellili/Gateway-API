# Gateway API / NGINX Gateway Fabric (NGF)

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
ansible-playbook playbook-crds.yml
ansible-playbook playbook-certificates.yml
ansible-playbook playbook.yml
ansible-playbook playbook-gateway-system.yml
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

| Playbook | Rôle |
|---|---|
| `playbook-crds.yml` | Installe les 8 CRDs Gateway API standard (channel v1.5.1), lues depuis un manifeste brut GitHub (pas de chart Helm pour les CRDs) |
| `playbook-certificates.yml` | Crée les certificats internes `server-tls` / `agent-tls` (communication control plane ↔ agent NGINX), via cert-manager et `internal-ca-issuer` |
| `playbook.yml` | Déploie le chart Helm NGF (control plane), avec `certGenerator.enable: false` puisque les certificats sont déjà gérés par cert-manager |
| `playbook-gateway-system.yml` | Crée le namespace `gateway-system`, les `Certificate` applicatifs (harbor-tls, argocd-tls, grafana-tls), et `platform-gateway` avec un Listener HTTPS par application |

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
