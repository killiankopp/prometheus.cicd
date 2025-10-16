# Prometheus + AlertManager GitOps Deployment

Déploiement production-like de Prometheus et AlertManager sur cluster k3d avec GitOps (ArgoCD).

## Architecture

- **Namespace**: `prometheus`
- **Domains**: 
  - `prometheus.amazone.lan` (Prometheus HTTPS)
  - `alertmanager.amazone.lan` (AlertManager HTTPS)
- **Storage**: PVC 10Gi (Prometheus) + 2Gi (AlertManager)
- **Ingress**: NGINX avec TLS automatique
- **GitOps**: ArgoCD pour déploiement zero-downtime

## Structure

```text
prometheus.cicd/
├── README.md
├── argocd/
│   ├── app.yaml                     # Application ArgoCD Prometheus
│   └── alertmanager-app.yaml        # Application ArgoCD AlertManager
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
- **Port-forward Prometheus**: `kubectl port-forward -n prometheus svc/prometheus 9090:9090`
- **Port-forward AlertManager**: `kubectl port-forward -n prometheus svc/alertmanager 9093:9093`

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

## Commandes utiles

```bash
# Test des charts Helm localement
helm template prometheus ./helm/prometheus
helm template alertmanager ./helm/alertmanager

# Validation des charts
helm lint ./helm/prometheus
helm lint ./helm/alertmanager

# Installation manuelle (bypass ArgoCD)
helm install prometheus ./helm/prometheus -n prometheus --create-namespace
helm install alertmanager ./helm/alertmanager -n prometheus

# Désinstallation complète
kubectl delete application prometheus alertmanager -n argocd
kubectl delete namespace prometheus

# Test des alertes
curl -X POST https://alertmanager.amazone.lan/api/v1/alerts \
  -H "Content-Type: application/json" \
  -d '[{"labels":{"alertname":"TestAlert","severity":"warning"}}]'
```