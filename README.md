# TP15 : Packager une application Kubernetes avec Helm et Kustomize

Master DevOps & Cloud Computing, ISET Tozeur. Application de démonstration : `traefik/whoami`, déployée sur le cluster K3s du TP14.

## Structure du dépôt

```
chart-starter/whoami/        Chart Helm complété (Chart.yaml, values.yaml, values.schema.json, templates/)
kustomize-starter/base/      Base Kustomize (deployment, service, configMapGenerator)
kustomize-starter/overlays/  Overlays dev, staging, prod
values-dev.yaml              Valeurs Helm pour dev
values-prod.yaml             Valeurs Helm pour prod
whoami-0.1.0.tgz, 0.1.1.tgz  Archives du chart (helm package)
captures/                    Captures d'écran des livrables
```

## Partie 1 : Chart public (podinfo)

```bash
helm repo add podinfo https://stefanprodan.github.io/podinfo && helm repo update
helm install demo podinfo/podinfo -n helm-demo --create-namespace --set replicaCount=2
helm get values demo -n helm-demo
helm uninstall demo -n helm-demo
```

**Où Helm stocke-t-il l'historique d'une release ?** Dans des Secrets Kubernetes de type `helm.sh/release.v1`, nommés `sh.helm.release.v1.<release>.v<révision>`, dans le namespace de la release. Il y a un Secret par révision.

## Partie 2 : Chart `whoami`

- `Chart.yaml` : `version` est la version du **chart** (à incrémenter à chaque modification du chart) ; `appVersion` est la version de l'**application** (tag d'image par défaut : `.Values.image.tag | default .Chart.AppVersion`).
- `deployment.yaml` : boucle `range` sur `.Values.env`, `readinessProbe` et `livenessProbe` sur `/health`.
- `service.yaml` : Service entièrement paramétré (nom, labels, type, port) avec `targetPort: http`.
- `ingress.yaml` : généré seulement si `ingress.enabled` vaut `true`.
- `values.schema.json` (défi) : valide les types de `values.yaml`.

Validation :

```bash
helm lint chart-starter/whoami
helm template test chart-starter/whoami --set ingress.enabled=true
helm install test chart-starter/whoami --dry-run=server -n tp15 --create-namespace
```

Captures : `captures/helm-lint-template.png`.

## Partie 3 : Cycle de vie d'une release

```bash
helm install whoami chart-starter/whoami -n tp15 --create-namespace -f values-dev.yaml
helm upgrade whoami chart-starter/whoami -n tp15 -f values-dev.yaml --set replicaCount=3
helm upgrade whoami chart-starter/whoami -n tp15 -f values-dev.yaml --set image.tag=v9.9.9   # ImagePullBackOff
helm history whoami -n tp15
helm rollback whoami 2 -n tp15
helm history whoami -n tp15
```

- **Rollback** : il ne « supprime » pas les révisions, il crée une **nouvelle révision** qui reprend la configuration de la révision ciblée. L'historique reste donc complet.
- **Upgrade sans `--atomic`** : Helm n'attend pas que les pods soient prêts, la release est marquée `deployed` alors que le nouveau pod est en `ImagePullBackOff` (les anciens pods continuent à servir grâce au rolling update).
- **Upgrade avec `--atomic --timeout 60s`** : Helm attend que les ressources soient prêtes ; au bout de 60 s (`context deadline exceeded`), il marque la révision `failed` et fait **automatiquement un rollback** vers la dernière révision stable. La release n'est jamais laissée dans un état cassé.
- **Packaging** : `helm package chart-starter/whoami` produit `whoami-<version>.tgz`. Le nom de l'archive suit la `version` du chart (pas `appVersion`) : passer de 0.1.0 à 0.1.1 donne `whoami-0.1.1.tgz`.

Captures : `captures/helm-history.png`, `captures/helm-atomic.png`, `captures/helm-schema.png`.

## Partie 4 : Kustomize, base et overlays

| Overlay | Namespace | Replicas | WHOAMI_NAME | Particularités |
|---|---|---|---|---|
| dev | tp15-dev | 1 | dev | LOG_LEVEL modifié pour l'exercice |
| staging | tp15-staging | 2 | staging | |
| prod | tp15-prod | 3 | prod | image `v1.10.1` (champ `images`), `limits.memory` 128Mi (patch), `readinessProbe` (patch stratégique depuis un fichier) |

```bash
kubectl apply -k kustomize-starter/overlays/dev
kubectl apply -k kustomize-starter/overlays/staging
kubectl apply -k kustomize-starter/overlays/prod
kubectl get deploy -A -l app.kubernetes.io/part-of=tp15
diff <(kubectl kustomize kustomize-starter/overlays/staging) <(kubectl kustomize kustomize-starter/overlays/prod)
```

**Pourquoi les Pods sont-ils recréés automatiquement quand on modifie `LOG_LEVEL` (contrairement au TP14) ?** Le `configMapGenerator` ajoute au nom de la ConfigMap un suffixe de hash calculé à partir de son contenu. Quand `LOG_LEVEL` change, le hash change, donc le nom change, et Kustomize met à jour la référence dans le Deployment (`envFrom.configMapRef.name`). Le template du Pod est alors modifié, ce qui déclenche un rolling update. Au TP14, la ConfigMap gardait un nom fixe et était modifiée sur place : le Deployment restait identique, donc les pods n'étaient pas recréés (il fallait un `kubectl rollout restart`). Constaté : `whoami-config-6t5d5gdccm` remplacée par `whoami-config-b54cmt8975` et nouveau pod créé. Note : l'ancienne ConfigMap reste présente, car `kubectl apply -k` ne supprime pas les orphelins sans `--prune`.

Captures : `captures/kustomize-envs.png`, `captures/kustomize-diff.png`.

## Partie 5 : Comparaison Helm / Kustomize

| Critère | Helm | Kustomize |
|---|---|---|
| Mécanisme (template / patch) | Templates Go + `values.yaml` : les manifests sont générés | Patches et overlays appliqués sur du YAML valide, sans moteur de template |
| Gestion des releases et rollback | Oui : releases versionnées (Secrets `sh.helm.release.v1`), `helm history`, `helm rollback`, `--atomic` | Non : pas de notion de release ni d'historique ; retour arrière via Git ou `kubectl rollout undo` |
| Distribution d'un logiciel à des tiers | Très adaptée : charts versionnés, dépôts, registres OCI, `helm install` | Peu adaptée : on partage des dossiers ou des dépôts Git, sans versionnement de paquet |
| Lisibilité du YAML final | Moyenne : templates avec `{{ }}` et `nindent`, il faut `helm template` pour voir le résultat | Bonne : la base est du YAML standard, `kubectl kustomize` montre le rendu |
| Intégré à kubectl | Non : binaire séparé à installer | Oui : `kubectl apply -k`, `kubectl kustomize` |
| Support par Argo CD | Oui, natif (charts, values, dépôts Helm et OCI) | Oui, natif (détection de `kustomization.yaml`) |

**Quand utiliser les deux ensemble ?** Quand on veut personnaliser un chart tiers sans forker ses templates : Kustomize rend le chart avec le champ `helmCharts:` (et ses values), puis applique par-dessus des patches propres à chaque environnement (ressources, labels, sécurité, namespace). On garde la distribution et le versionnement de Helm, et la personnalisation par overlays de Kustomize. Rendu avec `kubectl kustomize --enable-helm`.

## Défi

- `values.schema.json` : réalisé. `helm lint chart-starter/whoami --set replicaCount=trois` échoue avec `got string, want integer` (capture `captures/helm-schema.png`).
- Publication OCI vers Harbor (`helm push ... oci://harbor.lab.local/devsecops`) : non réalisée (Harbor du TP12 non fonctionnel sur la VM, conflit de ports avec K3s/Jenkins).

## Livrables

| Fichier | Statut |
|---|---|
| `captures/helm-lint-template.png` | fourni |
| `captures/helm-history.png` | fourni |
| `captures/helm-atomic.png` | fourni |
| `captures/kustomize-envs.png` | fourni |
| `captures/kustomize-diff.png` | fourni |
| `captures/helm-schema.png` | fourni (défi) |
| `captures/helm-oci.png` | non réalisé (défi optionnel) |
| Chart complété, `values-dev.yaml`, `values-prod.yaml`, overlays, tableau comparatif | dans ce dépôt |
