# Harbor (registre d'images + dépôt Helm privé)

Ce composant installe Harbor (`goharbor/harbor-helm`) — registre Docker, dépôt de charts Helm, scanner de vulnérabilités Trivy — et l'expose derrière la `Gateway` partagée (`platform-gateway`) livrée par le composant Gateway API/NGF.

## Architecture

Cette plateforme repose sur trois rôles distincts :

- **VM Mirror** — héberge le registre Docker (images) et le dépôt Helm (charts `.tgz`). C'est une source, rien ne s'y exécute.
- **VM PIC** — lance les playbooks Ansible (`ansible-playbook`), pilote le cluster via `kubectl`/`helm`. Contient les projets `bundle/` + `deploy-<composant>/`.
- **Cluster K8s** — exécute les workloads, doit pouvoir joindre le Mirror pour tirer images et charts.

**Point important** : Harbor n'a **aucune CRD propre** (contrairement à cert-manager, MetalLB ou Prometheus Operator) — il n'y a donc pas de `playbook-crds.yml` pour ce composant. Sur un cluster disposant d'un CSI (Rook-Ceph ou équivalent), la structure standard à deux playbooks (Helm release + route) suffit. Sur un cluster sans CSI, un troisième playbook (`playbook-local-storage.yml`) est requis en amont — voir section *Pratique : déploiement sans CSI* plus bas.

**Piège déjà rencontré — collision d'objet avec le chart** : par défaut, le chart Harbor génère lui-même un `HTTPRoute` (`expose.type: route`) sous le nom `<release>-route`, soit `harbor-route` — exactement le nom que ce composant utilise pour son propre `HTTPRoute` géré par Ansible. Les deux entrent en collision : Helm écrase silencieusement l'objet à chaque `upgrade`, ou échoue avec une erreur de possession de ressource. **`expose.type` doit rester sur `clusterIP`** dans `values-harbor.yaml.j2` — le chart se contente alors de créer des `Service` ClusterIP classiques pour `core`/`portal`, et seul le `HTTPRoute` géré par ce composant route le trafic entrant.

**Piège déjà rencontré — ServiceMonitor invisible malgré `enabled: true`** : activer `metrics.serviceMonitor.enabled: true` dans le chart ne suffit pas à faire fonctionner le monitoring. Le `ServiceMonitor` généré par défaut porte le label `release: harbor` (le nom de la release Helm), alors que le `serviceMonitorSelector` du CR `Prometheus` attend `release: kube-prometheus-stack`. Sans ce label exact, Prometheus ignore silencieusement le `ServiceMonitor` — la ressource existe côté Kubernetes, mais n'apparaît jamais dans `Status → Targets` de Prometheus, sans erreur visible nulle part. Le chart expose heureusement `metrics.serviceMonitor.additionalLabels`, utilisé dans le template pour forcer `release: kube-prometheus-stack`. **Toujours vérifier après déploiement** (`kubectl get servicemonitor harbor -n harbor -o jsonpath='{.metadata.labels}'` + `up{namespace="harbor"}` dans Prometheus), ne jamais se fier au flag `enabled: true` seul.

**Piège déjà rencontré — `kubernetes.core.k8s` sans kubeconfig explicite** : sur un nouveau cluster, une tâche `kubernetes.core.k8s`/`k8s_info` qui ne précise pas `kubeconfig:` dépend du fichier par défaut de l'utilisateur système (`~/.kube/config`) sur l'hôte où s'exécute réellement la tâche — un échec `Could not create API client: Invalid kube-config file. No configuration found.` signifie typiquement que ce fichier n'a jamais été copié/lié pour cet utilisateur sur ce nœud précis (`/etc/kubernetes/admin.conf` existe, mais `~/.kube/config` non). Deux correctifs possibles : copier le kubeconfig au bon endroit (`mkdir -p ~/.kube && cp /etc/kubernetes/admin.conf ~/.kube/config`), ou — préférable, indépendant de l'environnement de l'utilisateur exécutant Ansible — passer `kubeconfig: "{{ k8s_kubeconfig_path }}"` explicitement sur chaque tâche `kubernetes.core.k8s*`, avec `k8s_kubeconfig_path: "/etc/kubernetes/admin.conf"` centralisé dans `group_vars/all.yml`.

Le `Certificate harbor-tls` est géré ailleurs, centralisé dans `gateway-system` avec ceux de Grafana et ArgoCD.

---

## 1️⃣ Déploiement

Depuis la VM PIC :

```bash
cd deploy-harbor

# Uniquement si le cluster n'a pas de CSI (voir section Pratique plus bas)
ansible-playbook -i inventory.ini playbook-local-storage.yml

ansible-playbook -i inventory.ini playbook-helm-release.yml -e "harbor_storage_class=<storage-class-du-cluster>"
ansible-playbook -i inventory.ini playbook-route.yml
```

**Ordre strict à respecter** :

```
(playbook-local-storage.yml)   ← seulement si pas de CSI : StorageClass manual + 5 PV
        ↓
playbook-helm-release.yml
        ↓ (les Service harbor-core / harbor-portal doivent exister
           avant que le HTTPRoute qui les référence ne soit créé)
playbook-route.yml
```

> **Prérequis externe** : `platform-gateway` (namespace `gateway-system`) et le `Certificate harbor-tls` doivent déjà exister avant `playbook-route.yml` — tous deux créés par le composant Gateway API/NGF, pas par celui-ci.

> **StorageClass** : pas de valeur par défaut fiable dans `group_vars/all.yml` (`harbor_storage_class` y est un placeholder, `ceph-rbd` ou `ceph-block` selon le cluster réel — nom exact toujours à confirmer via `kubectl get sc` avant tout déploiement, jamais deviner). Sur un cluster sans CSI (`local-path` absent, pas de Rook-Ceph), voir la section *Pratique : déploiement sans CSI* plus bas.

---

## 2️⃣ Vérifications

**Pods** :

```bash
kubectl get pods -n harbor
```

Attendu, tous `Running` : `harbor-core-*` (1/1), `harbor-portal-*` (1/1), `harbor-jobservice-*` (1/1), `harbor-registry-*` (2/2 — conteneurs `registry` + `registryctl`), `harbor-database-0` (1/1), `harbor-redis-0` (1/1), `harbor-trivy-0` (1/1), `harbor-exporter-*` (1/1).

**Stockage persistant** (5 composants avec état) :

```bash
kubectl get pvc -n harbor
```

Attendu, tous `Bound` : `harbor-registry`, `harbor-jobservice`, `database-data-harbor-database-0`, `data-harbor-redis-0`, `data-harbor-trivy-0`. Ces noms de PVC sont fixés par le chart (PVC simples pour `registry`/`jobservice`, `volumeClaimTemplates` pour les 3 StatefulSet) — confirmés via `helm template harbor ./harbor --set persistence.enabled=true | grep -A3 "kind: PersistentVolumeClaim"`, jamais devinés.

**Images réellement utilisées** (toutes doivent pointer vers le registre Mirror) :

```bash
kubectl get pods -n harbor -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.containers[*]}  {.name}: {.image}{"\n"}{end}{end}'
```

10 images au total, réparties sur 8 composants : `nginx-photon` (proxy interne au chart, non utilisé en mode `clusterIP`), `harbor-portal`, `harbor-core`, `harbor-jobservice`, `registry-photon` + `harbor-registryctl` (2 images pour le composant registry), `trivy-adapter-photon`, `harbor-db`, `redis-photon`, `harbor-exporter`.

**ServiceMonitor scrapé par Prometheus** (voir piège ci-dessus — ne pas se fier uniquement à l'existence de la ressource) :

```bash
kubectl get servicemonitor harbor -n harbor -o jsonpath='{.metadata.labels}' ; echo
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090 &
curl -s 'http://localhost:9090/api/v1/query?query=up%7Bnamespace%3D%22harbor%22%7D' | python3 -m json.tool
kill %1
```

Attendu : le label `release: kube-prometheus-stack` présent, et au moins 4 séries `up` à `1` (jobservice, registry, core, exporter).

**HTTPRoute** :

```bash
kubectl describe httproute harbor-route -n harbor
```

Attendu : `Accepted: True`, `ResolvedRefs: True` sur la condition de parent.

---

## 3️⃣ Test fonctionnel

Test end-to-end sur le hostname Harbor, via `platform-gateway` :

```bash
curl -k --resolve harbor.dev.local:443:<IP_PLATFORM_GATEWAY> https://harbor.dev.local/api/v2.0/systeminfo
```

Attendu : réponse JSON (`auth_mode`, `self_registration`, etc.). Ajouter `-v` pour inspecter la chaîne de certificats — elle doit correspondre au `Certificate harbor-tls` créé dans `gateway-system`, signé par `internal-ca-issuer`.

Connexion au portail : `admin` / valeur de `harbor_admin_password` (défaut du chart, `Harbor12345` — **à changer après le premier login**, jamais laissé tel quel en prod).

---

## Contenu des playbooks

**`playbook-helm-release.yml`**

- *Create harbor namespace* : crée le namespace `harbor`
- *Add internal Helm repository* : ajoute le dépôt Helm pointant vers la VM Mirror
- *Render values* : génère `values-harbor.yaml` à partir du template — overrides des 10 images, `expose.type: clusterIP` (jamais `route`, voir piège plus haut), `persistence.persistentVolumeClaim.<composant>.storageClass` (paramétrable par `-e`), config Trivy offline (`skipUpdate`/`skipJavaDBUpdate`/`offlineScan` à `true` tant que la vuln DB n'est pas livrée manuellement), `metrics.serviceMonitor.additionalLabels.release: kube-prometheus-stack`
- *Deploy via Helm* : installe/upgrade le release `harbor`
- *Verify pods Running* : liste et affiche la phase de chaque pod du namespace

**`playbook-route.yml`**

- *Create Harbor HTTPRoute* : crée le `HTTPRoute harbor-route` dans `harbor`, `parentRefs` vers `platform-gateway`/`gateway-system`, deux règles de routage par chemin — `/api/`, `/service/`, `/v2/`, `/c/` → backend `harbor-core:80` ; `/` (tout le reste) → backend `harbor-portal:80`. Ne crée ni `Certificate` ni `Gateway`.

---

## Variables clés (`group_vars/all.yml`)

```yaml
# --- Chart / release ---
harbor_chart_version: "1.19.1"
harbor_app_version: "v2.15.1"
harbor_namespace: "harbor"
harbor_release_name: "harbor"

# --- Registre et dépôt Helm internes ---
image_registry: "192.168.1.20:5000/harbor"
harbor_helm_repo_name: "internal-helm-repo"
harbor_helm_repo_url: "http://192.168.1.20:8080"

# --- Images (10 au total) ---
harbor_images:
  nginx:       { repository: "{{ image_registry }}/nginx-photon",          tag: "v2.15.1" }
  portal:      { repository: "{{ image_registry }}/harbor-portal",         tag: "v2.15.1" }
  core:        { repository: "{{ image_registry }}/harbor-core",           tag: "v2.15.1" }
  jobservice:  { repository: "{{ image_registry }}/harbor-jobservice",     tag: "v2.15.1" }
  registry:    { repository: "{{ image_registry }}/registry-photon",       tag: "v2.15.1" }
  registryctl: { repository: "{{ image_registry }}/harbor-registryctl",    tag: "v2.15.1" }
  trivy:       { repository: "{{ image_registry }}/trivy-adapter-photon",  tag: "v2.15.1" }
  database:    { repository: "{{ image_registry }}/harbor-db",             tag: "v2.15.1" }
  redis:       { repository: "{{ image_registry }}/redis-photon",          tag: "v2.15.1" }
  exporter:    { repository: "{{ image_registry }}/harbor-exporter",       tag: "v2.15.1" }

# --- Stockage (placeholder, override via -e "harbor_storage_class=...") ---
harbor_storage_class: "ceph-rbd"   # à confirmer sur chaque cluster cible, jamais deviner
harbor_registry_pvc_size: "5Gi"        # référence prod observée : 300Gi
harbor_jobservice_pvc_size: "1Gi"
harbor_database_pvc_size: "1Gi"        # référence prod observée : 8Gi
harbor_redis_pvc_size: "1Gi"           # référence prod observée : 12Gi
harbor_trivy_pvc_size: "5Gi"

# --- Exposition (via platform-gateway) ---
harbor_external_url: "https://harbor.dev.local"
harbor_hostname: "harbor.dev.local"

# --- Trivy (mode offline préparé) ---
trivy_enabled: true
trivy_skip_update: true
trivy_skip_java_db_update: true
trivy_offline_scan: true

# --- Identifiants (défauts du chart — à changer avant prod) ---
harbor_admin_password: "Harbor12345"
harbor_db_password: "changeit"

# --- Monitoring ---
monitoring_enabled: true

# --- Gateway partagée (référence au composant Gateway API/NGF) ---
gateway_system_namespace: "gateway-system"
platform_gateway_name: "platform-gateway"
```

> **Note sur Trivy** — le scanner ne se contente pas d'une image container : au premier scan réel, il télécharge sa base de vulnérabilités (`trivy-db` + `trivy-java-db`) depuis GitHub/GHCR, indépendamment du registre Mirror. En air-gapped, `trivy_enabled: true` seul ne suffit pas pour un scan fonctionnel : soit désactiver Trivy (`trivy_enabled: false`) si le scan n'est pas un besoin immédiat, soit pré-télécharger la vuln DB sur une machine avec accès Internet (`trivy image --download-db-only`) et la monter manuellement dans `/home/scanner/.cache/trivy/db/`. Le composant tourne (`Running`) sans cette DB — il échoue seulement au moment d'un scan réel, pas au démarrage.

---

## Pratique : déploiement sans CSI (StorageClass `manual`, PV locaux)

Sur un cluster sans CSI répliqué disponible (pas de Rook-Ceph, pas même `local-path-provisioner`), le stockage des 5 composants avec état est fourni via des `PersistentVolume` de type `local`, créés manuellement par `playbook-local-storage.yml` — un playbook dédié, à exécuter **avant** `playbook-helm-release.yml`.

**Limites à accepter** : chaque volume est épinglé à un nœud précis — si ce nœud tombe, les 5 pods concernés restent `Pending` indéfiniment (pas de failover). Les pods sans état (`core`, `portal`, `exporter`) restent librement schedulables partout, seuls les 5 composants avec PVC sont contraints.

**Variables dédiées (`group_vars/all.yml`)** :
```yaml
harbor_local_storage_node: "worker-lumio9"      # nœud hébergeant les 5 volumes
k8s_kubeconfig_path: "/etc/kubernetes/admin.conf"  # voir piège kubeconfig ci-dessus

harbor_local_volumes:
  - pv_name: harbor-registry-pv
    pv_claim_name: harbor-registry
    pv_size: "{{ harbor_registry_pvc_size }}"
    pv_path: /harbor/harbor-registry
    pv_node: "{{ harbor_local_storage_node }}"
  - pv_name: harbor-jobservice-pv
    pv_claim_name: harbor-jobservice
    pv_size: "{{ harbor_jobservice_pvc_size }}"
    pv_path: /harbor/harbor-jobservice
    pv_node: "{{ harbor_local_storage_node }}"
  - pv_name: harbor-database-pv
    pv_claim_name: database-data-harbor-database-0
    pv_size: "{{ harbor_database_pvc_size }}"
    pv_path: /harbor/harbor-database
    pv_node: "{{ harbor_local_storage_node }}"
  - pv_name: harbor-redis-pv
    pv_claim_name: data-harbor-redis-0
    pv_size: "{{ harbor_redis_pvc_size }}"
    pv_path: /harbor/harbor-redis
    pv_node: "{{ harbor_local_storage_node }}"
  - pv_name: harbor-trivy-pv
    pv_claim_name: data-harbor-trivy-0
    pv_size: "{{ harbor_trivy_pvc_size }}"
    pv_path: /harbor/harbor-trivy
    pv_node: "{{ harbor_local_storage_node }}"
```

Le `claimRef` de chaque volume pointe vers le nom de PVC exact attendu par le chart (voir section Vérifications ci-dessus pour la liste) — sans `claimRef`, le binding se fait par capacité compatible plutôt que par composant voulu, avec risque de mauvais appariement silencieux.

**`playbook-local-storage.yml`** — trois tâches :

- *Créer les répertoires de stockage local* : `ansible.builtin.file` en boucle sur les 5 chemins, **délégué à `{{ harbor_local_storage_node }}`** (`delegate_to`, jamais le nom du nœud en dur — le nœud peut différer d'un cluster à l'autre)
- *Rendu + application de la StorageClass `manual`* (`provisioner: kubernetes.io/no-provisioner`, `volumeBindingMode: WaitForFirstConsumer`) — rendu délégué à `localhost`, application sur le control-plane avec `kubeconfig: "{{ k8s_kubeconfig_path }}"` explicite
- *Rendu + application des 5 `PersistentVolume`*, en boucle sur `harbor_local_volumes`, même schéma rendu/application

Avant d'exécuter sur un nouveau nœud, préparer le disque dédié :
```bash
sudo mkfs.ext4 /dev/sdb
sudo mkdir -p /harbor
sudo mount /dev/sdb /harbor
echo "UUID=$(sudo blkid -s UUID -o value /dev/sdb)  /harbor  ext4  defaults  0  2" | sudo tee -a /etc/fstab
```
(la création des sous-dossiers `/harbor/harbor-<composant>` est automatisée par le playbook lui-même, pas besoin de les créer à la main)

**Exécution** :
```bash
ansible-playbook -i inventory.ini playbook-local-storage.yml
ansible-playbook -i inventory.ini playbook-helm-release.yml -e "harbor_storage_class=manual"
```

**Vérifier le binding** — les 5 PV passent de `Available` à `Bound` uniquement au moment où Harbor crée réellement ses PVC (comportement `WaitForFirstConsumer`) :
```bash
kubectl get pv
kubectl get pvc -n harbor
```
