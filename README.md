# Prometheus + AlertManager + Blackbox Exporter GitOps Deployment

Déploiement production-like de la stack Prometheus complète sur cluster k3d avec GitOps (ArgoCD).

## Architecture

- **Namespace**: `prometheus`
- **Domains**:
  - `prometheus.amazone.lan` (Prometheus HTTPS)
  - `alertmanager.amazone.lan` (AlertManager HTTPS)
  - `blackbox.amazone.lan` (Blackbox Exporter HTTPS - optionnel)
- **Storage**: PVC 10Gi (Prometheus) + 2Gi (AlertManager)
- **Ingress**: NGINX avec TLS automatique
- **GitOps**: ArgoCD pour déploiement zero-downtime
- **Monitoring**: HTTP/HTTPS, TCP, ICMP, DNS probes via Blackbox Exporter

## Structure

```text
prometheus.cicd/
├── README.md
├── argocd/
│   ├── app.yaml                     # Application ArgoCD Prometheus
│   ├── alertmanager-app.yaml        # Application ArgoCD AlertManager
│   └── blackbox-exporter-app.yaml  # Application ArgoCD Blackbox Exporter
└── helm/
    ├── prometheus/
    │   ├── Chart.yaml               # Métadonnées du chart
    │   ├── values.yaml              # Configuration Prometheus
    │   └── templates/
    │       ├── _helpers.tpl         # Fonctions utilitaires
    │       ├── configmap.yaml       # Configuration Prometheus
    │       ├── deployment.yaml      # Déploiement Prometheus
    │       ├── ingress.yaml         # Exposition HTTPS
    │       ├── pvc.yaml            # Stockage persistant
    │       ├── rules.yaml          # Règles d'alerte
    │       ├── secret.yaml         # Secrets auto-générés
    │       ├── service.yaml        # Service interne
    │       └── serviceaccount.yaml
    └── alertmanager/
        ├── Chart.yaml               # Métadonnées AlertManager
        ├── values.yaml              # Configuration AlertManager
        └── templates/
            ├── _helpers.tpl         # Fonctions utilitaires
            ├── configmap.yaml       # Configuration AlertManager
            ├── deployment.yaml      # Déploiement AlertManager
            ├── ingress.yaml         # Exposition HTTPS
            ├── pvc.yaml            # Stockage persistant
            ├── secret.yaml         # Secrets auto-générés
            ├── service.yaml        # Service interne
            └── serviceaccount.yaml
    └── blackbox-exporter/
        ├── Chart.yaml               # Métadonnées Blackbox Exporter
        ├── values.yaml              # Configuration Blackbox Exporter
        └── templates/
            ├── _helpers.tpl         # Fonctions utilitaires
            ├── configmap.yaml       # Configuration modules de probe
            ├── deployment.yaml      # Déploiement Blackbox Exporter
            ├── ingress.yaml         # Exposition HTTPS (optionnel)
            ├── service.yaml         # Service interne
            └── serviceaccount.yaml
```

## Prérequis

- Cluster k3d fonctionnel
- ArgoCD installé et configuré
- NGINX Ingress Controller
- cert-manager avec ClusterIssuer `local-amazone`
- DNS: `prometheus.amazone.lan` → IP cluster

## Déploiement

### 1. Créer les applications ArgoCD

```bash
# Déployer Prometheus
kubectl apply -f argocd/app.yaml

# Déployer AlertManager
kubectl apply -f argocd/alertmanager-app.yaml

# Déployer Blackbox Exporter
kubectl apply -f argocd/blackbox-exporter-app.yaml
```

### 2. Vérifier le déploiement

```bash
# Statut des applications ArgoCD
kubectl get application -n argocd

# Statut des pods
kubectl get pods -n prometheus

# Statut des PVC
kubectl get pvc -n prometheus

# Certificats TLS
kubectl get certificate -n prometheus
```

### 3. Accès aux services

- **Prometheus**: `https://prometheus.amazone.lan`
- **AlertManager**: `https://alertmanager.amazone.lan`
- **Blackbox Exporter**: `https://blackbox.amazone.lan` (si ingress activé)
- **Port-forward Prometheus**: `kubectl port-forward -n prometheus svc/prometheus 9090:9090`
- **Port-forward AlertManager**: `kubectl port-forward -n prometheus svc/alertmanager 9093:9093`
- **Port-forward Blackbox**: `kubectl port-forward -n prometheus svc/blackbox-exporter 9115:9115`

## Configuration

### Variables importantes dans `values.yaml`

```yaml
image:
  repository: prom/prometheus
  tag: "v2.47.0"

persistence:
  size: 10Gi

ingress:
  hostname: prometheus.amazone.lan

resources:
  requests:
    cpu: 250m
    memory: 1Gi
  limits:
    cpu: 500m
    memory: 2Gi
```

### Personnalisation

1. **Modifier la taille du stockage**:
   ```yaml
   persistence:
     size: 50Gi
   ```

2. **Changer le domaine**:
   ```yaml
   ingress:
     hostname: monitoring.example.com
   ```

3. **Ajuster les ressources**:
   ```yaml
   resources:
     requests:
       memory: 2Gi
     limits:
       memory: 4Gi
   ```

## Opérations

### Mise à jour

1. Modifier `values.yaml`
2. Commit & push vers le repo Git
3. ArgoCD détecte et applique automatiquement (ou sync manuel)

### Surveillance

```bash
# Logs du pod Prometheus
kubectl logs -n prometheus deployment/prometheus -f

# Événements du namespace
kubectl get events -n prometheus --sort-by='.lastTimestamp'

# Utilisation des ressources
kubectl top pods -n prometheus
```

### Sauvegarde

```bash
# Snapshot des données Prometheus
kubectl exec -n prometheus deployment/prometheus -- \
  wget -qO- http://localhost:9090/api/v1/admin/tsdb/snapshot
```

### Dépannage

#### Pod en erreur
```bash
kubectl describe pod -n prometheus -l app.kubernetes.io/name=prometheus
kubectl logs -n prometheus deployment/prometheus
```

#### Problème de certificat
```bash
kubectl describe certificate prometheus-tls -n prometheus
kubectl get certificaterequest -n prometheus
```

#### Ingress inaccessible
```bash
kubectl get ingress -n prometheus
kubectl describe ingress prometheus -n prometheus
```

## Limitations & Avertissements

⚠️ **Configuration par défaut**:
- Rétention des données: 15 jours
- Pas de haute disponibilité (1 replica)
- Pas d'alertes configurées (AlertManager séparé)

⚠️ **Sécurité**:
- Secrets auto-générés à chaque déploiement
- Pas d'authentification activée par défaut
- Accès admin API activé (pour lifecycle management)

⚠️ **Performance**:
- Configuration basique pour environnement de développement
- Surveiller l'utilisation mémoire en production

## URLs utiles

- **Interface Prometheus**: <https://prometheus.amazone.lan>
- **Interface AlertManager**: <https://alertmanager.amazone.lan>
- **Métriques Prometheus**: <https://prometheus.amazone.lan/metrics>
- **Configuration Prometheus**: <https://prometheus.amazone.lan/config>
- **Targets Prometheus**: <https://prometheus.amazone.lan/targets>
- **Alerts**: <https://prometheus.amazone.lan/alerts>
- **Blackbox Probes**: <http://localhost:9115/probe?module=http_2xx&target=https://example.com> (via port-forward)

## Alerting

### Règles d'alerte incluses

- **PrometheusTargetMissing**: Détecte les targets indisponibles
- **PrometheusConfigurationReloadFailure**: Échec de rechargement de config
- **PrometheusTooManyRestarts**: Trop de redémarrages

### Configuration AlertManager

Par défaut, AlertManager est configuré avec :

- **Webhook générique** pour les tests
- **Grouping** par nom d'alerte
- **Inhibition** des alertes warning en cas de critical

### Personnaliser les notifications

Modifier `helm/alertmanager/values.yaml` :

```yaml
alertmanager:
  config:
    global:
      smtp_smarthost: 'smtp.example.com:587'
      smtp_from: 'alerts@example.com'
    receivers:
      - name: 'email-notifications'
        email_configs:
          - to: 'admin@example.com'
            subject: 'Alerte {{ .GroupLabels.alertname }}'
```

## Blackbox Exporter

### Modules de probe configurés

- **http_2xx**: Tests HTTP/HTTPS avec codes de statut 200-399
- **http_post_2xx**: Tests HTTP POST avec JSON body
- **tcp_connect**: Tests de connexion TCP
- **icmp**: Tests de ping ICMP
- **dns_udp**: Tests de résolution DNS
- **ssh_banner**: Tests de bannière SSH

### Targets surveillées par défaut

- **HTTP/HTTPS**: Services Prometheus et AlertManager
- **ICMP**: Serveurs DNS publics (8.8.8.8, 1.1.1.1)
- **Personnalisables** via `values.yaml`

### Personnaliser les targets

Modifier `helm/blackbox-exporter/values.yaml` :

```yaml
monitoring:
  targets:
    http:
      - name: "mon-site"
        url: "https://mon-site.com"
        module: "http_2xx"
    icmp:
      - name: "mon-serveur"
        target: "192.168.1.100"
        module: "icmp"
```

### Règles d'alerte Blackbox

- **BlackboxProbeFailed**: Échec de probe pendant 5min
- **BlackboxSlowProbe**: Probe trop lente (>1s)
- **BlackboxProbeHttpFailure**: Code HTTP d'erreur
- **BlackboxSslCertificateWillExpireSoon**: Certificat SSL expirant

## Commandes utiles

```bash
# Test des charts Helm localement
helm template prometheus ./helm/prometheus
helm template alertmanager ./helm/alertmanager
helm template blackbox-exporter ./helm/blackbox-exporter

# Validation des charts
helm lint ./helm/prometheus
helm lint ./helm/alertmanager
helm lint ./helm/blackbox-exporter

# Installation manuelle (bypass ArgoCD)
helm install prometheus ./helm/prometheus -n prometheus --create-namespace
helm install alertmanager ./helm/alertmanager -n prometheus
helm install blackbox-exporter ./helm/blackbox-exporter -n prometheus

# Désinstallation complète
kubectl delete application prometheus alertmanager blackbox-exporter -n argocd
kubectl delete namespace prometheus

# Test des alertes
curl -X POST https://alertmanager.amazone.lan/api/v1/alerts \
  -H "Content-Type: application/json" \
  -d '[{"labels":{"alertname":"TestAlert","severity":"warning"}}]'

# Test manuel d'une probe Blackbox
kubectl port-forward -n prometheus svc/blackbox-exporter 9115:9115 &
curl "http://localhost:9115/probe?module=http_2xx&target=https://example.com"
```