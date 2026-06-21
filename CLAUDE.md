# capi-proxmox-clusterclass

Repo "modèle" — ClusterClass CAPI+Proxmox réutilisable. Tiré via ArgoCD par le repo instance `../capi-bootstrap/`.

Remote : github.com/UnPoilTefal/capi-proxmox-clusterclass (public, branche `main`)

## Structure

```
kustomization.yaml          ← agrégateur racine (ns.yaml + kubeadm-ubuntu/)
ns.yaml                     ← namespace caprox-kubernetes-engine
kubeadm-ubuntu/             ← ClusterClass kubeadm + Ubuntu 26.04
  kustomization.yaml
  clusterclass.yaml         ← proxmox-clusterclass-v0.2.0 (variables + patches inline)
  templates/
    cni/                    ← HelmChartProxy Calico (label caprox.eu/cni: calico)
    csi/                    ← HelmChartProxy CCM + ClusterResourceSet credentials Proxmox
    svc-lb/                 ← ClusterResourceSet kube-vip service LB
    kubeadm-control-plane-template.yaml   ← proxmox-kubernetes-engine-v4 (IMMUTABLE)
    kubeadm-config-template.yaml
    proxmox-machine-template-cp.yaml      ← IMMUTABLE
    proxmox-machine-template-worker.yaml  ← IMMUTABLE
    proxmox-cluster-template.yaml
talos/                      ← (à venir) ClusterClass Talos
```

## Déploiement

ArgoCD (selfHeal: true) sur le mgmt cluster tire ce repo automatiquement après chaque push.

```bash
# Prévisualiser le rendu Kustomize complet
kubectl kustomize .

# Appliquer manuellement si nécessaire (hors ArgoCD)
kubectl kustomize . | kubectl apply -f - --context admin@capi-mgmt
```

## Repo instance lié

`../capi-bootstrap/` — contient les Cluster CR (`clusters/mgmt/cluster.yaml`, `clusters/app/cluster.yaml`) et le bootstrap.

## Infra cible

- Proxmox VE, provider CAPMOX v0.8.1
- Template VM : ID 9200 sur prox-node-03, TAG `v1-35-5` (Ubuntu 26.04, K8s v1.35.5)
- Namespace CAPI : `caprox-kubernetes-engine`
- Clusters gérés : `mgmt` (admin@capi-mgmt), `app` (admin@app)

## Règles importantes

**ProxmoxMachineTemplate est IMMUTABLE** (webhook CAPMOX).
Tout changement de spec dans `proxmox-machine-template-*.yaml` ou `kubeadm-control-plane-template.yaml`
exige un renommage du fichier et de la ressource (ex: `-v4` → `-v5`), puis mise à jour de la référence
dans `clusterclass.yaml`.

**Template CAPMOX découvert par TAG cluster-wide** : un seul template Proxmox par valeur de TAG.
Plusieurs templates avec le même TAG = erreur "found N VM templates".

**Ne pas mettre de valeurs d'instance** (IPs, nœuds Proxmox, secrets) dans ce repo — elles appartiennent
aux Cluster CR dans `../capi-bootstrap/clusters/`.

## Décisions techniques

- Variables et patches inline dans `clusterclass.yaml` (pas de fichiers séparés)
- MHC timeout 600s : Calico peut prendre >5min sur un nœud sans cache images
- CCM toleration `operator: Exists` pour `node.cloudprovider.kubernetes.io/uninitialized`
- `vip_interface: ""` → kube-vip v1.0.0 auto-détecte l'interface (validé sur Ubuntu 26.04 / ens18)
- Workers `replicas: 0` par défaut recommandé dans les Cluster CR (eviter contention CFS lock Proxmox)
