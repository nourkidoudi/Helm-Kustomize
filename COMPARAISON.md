# TP15 : Comparaison Helm / Kustomize

## Tableau comparatif

| Critère | Helm | Kustomize |
|---|---|---|
| Mécanisme (template / patch) | Templates Go + `values.yaml` : les manifests sont générés à partir de variables | Patches et overlays appliqués sur du YAML valide, sans moteur de template |
| Gestion des releases et rollback | Oui : releases versionnées (Secrets `sh.helm.release.v1`), `helm history`, `helm rollback`, `--atomic` | Non : pas de notion de release ni d'historique ; retour arrière via Git ou `kubectl rollout undo` |
| Distribution d'un logiciel à des tiers | Très adaptée : charts versionnés, dépôts Helm, registres OCI, `helm install` | Peu adaptée : on partage des dossiers ou des dépôts Git, sans versionnement de paquet |
| Lisibilité du YAML final | Moyenne : templates avec `{{ }}` et `nindent`, il faut `helm template` pour voir le résultat | Bonne : la base est du YAML standard, `kubectl kustomize` montre le rendu |
| Intégré à kubectl | Non : binaire séparé à installer | Oui : `kubectl apply -k` et `kubectl kustomize` |
| Support par Argo CD | Oui, natif (charts, values, dépôts Helm et OCI) | Oui, natif (détection de `kustomization.yaml`) |

## Quand utiliser les deux ensemble ?

Quand on veut personnaliser un chart tiers sans forker ses templates. Kustomize rend le chart avec le champ `helmCharts:` (et ses values), puis applique par-dessus des patches propres à chaque environnement (ressources, labels, sécurité, namespace). On garde ainsi la distribution et le versionnement de Helm, et la personnalisation par overlays de Kustomize.

Exemple de `kustomization.yaml` :

```yaml
helmCharts:
  - name: podinfo
    repo: https://stefanprodan.github.io/podinfo
    version: <version>
    releaseName: demo
    valuesInline:
      replicaCount: 2
patches:
  - path: patch-resources.yaml
```

Rendu avec :

```bash
kubectl kustomize --enable-helm
```

## Conclusion : quel outil choisir ?

- **Helm** : pour distribuer une application à d'autres équipes ou à des tiers, avoir des releases versionnées avec upgrade et rollback, et paramétrer fortement l'application.
- **Kustomize** : pour gérer ses propres applications sur plusieurs environnements (dev, staging, prod) avec un YAML lisible, des différences minimes entre environnements, et sans installer d'outil supplémentaire.
- **Les deux** : un chart tiers déployé et personnalisé par environnement, ou un pipeline GitOps (Argo CD) qui combine les deux.
