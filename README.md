# Prometheus GitOps Deployment

Déploiement production-like de Prometheus sur cluster k3d avec GitOps (ArgoCD).

## Architecture

- **Namespace**: `prometheus`
- **Domain**: `prometheus.amazone.lan` (HTTPS via cert-manager)
- **Storage**: PVC 10Gi (StorageClass par défaut)
- **Ingress**: NGINX avec TLS automatique
- **GitOps**: ArgoCD pour déploiement zero-downtime

## Structure

```
prometheus.cicd/
├── README.md
├── argocd/
│   └── app.yaml                 # Application ArgoCD
└── helm/
    └── prometheus/
        ├── Chart.yaml           # Métadonnées du chart
        ├── values.yaml          # Configuration par défaut
        └── templates/
            ├── _helpers.tpl     # Fonctions utilitaires
            ├── configmap.yaml   # Configuration Prometheus
            ├── deployment.yaml  # Déploiement principal
            ├── ingress.yaml     # Exposition HTTPS
            ├── pvc.yaml         # Stockage persistant
            ├── secret.yaml      # Secrets auto-générés
            ├── service.yaml     # Service interne
            └── serviceaccount.yaml
```

## Prérequis

- Cluster k3d fonctionnel
- ArgoCD installé et configuré
- NGINX Ingress Controller
- cert-manager avec ClusterIssuer `local-amazone`
- DNS: `prometheus.amazone.lan` → IP cluster

## Déploiement

### 1. Créer l'application ArgoCD

```bash
kubectl apply -f argocd/app.yaml
```

### 2. Vérifier le déploiement

```bash
# Statut de l'application ArgoCD
kubectl get application prometheus -n argocd

# Statut des pods
kubectl get pods -n prometheus

# Statut du PVC
kubectl get pvc -n prometheus

# Certificat TLS
kubectl get certificate -n prometheus
```

### 3. Accès au service

- **Local**: `https://prometheus.amazone.lan`
- **Port-forward**: `kubectl port-forward -n prometheus svc/prometheus 9090:9090`

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

- **Interface Prometheus**: https://prometheus.amazone.lan
- **Métriques**: https://prometheus.amazone.lan/metrics
- **Configuration**: https://prometheus.amazone.lan/config
- **Targets**: https://prometheus.amazone.lan/targets

## Commandes utiles

```bash
# Test du chart Helm localement
helm template prometheus ./helm/prometheus

# Validation du chart
helm lint ./helm/prometheus

# Installation manuelle (bypass ArgoCD)
helm install prometheus ./helm/prometheus -n prometheus --create-namespace

# Désinstallation complète
kubectl delete application prometheus -n argocd
kubectl delete namespace prometheus
```