# 🚀 Déploiement d'AWX sur K3s avec AWX Operator

> Guide pas-à-pas pour déployer **Ansible AWX** sur un cluster **K3s** via l'AWX Operator, avec stockage persistant et exposition via Traefik Ingress.

---

## 📋 Prérequis

- Un cluster **K3s** opérationnel
- `kubectl` configuré et fonctionnel
- `make` installé sur la machine
- `git` installé
- Accès Internet depuis le nœud (pour pull les images)
- Traefik installé en tant qu'Ingress Controller *(inclus par défaut dans K3s)*

---

## 📦 1. Cloner l'AWX Operator

```bash
git clone https://github.com/ansible/awx-operator.git
cd awx-operator
```

Lister les tags disponibles et choisir la version souhaitée :

```bash
git tag
git checkout tags/<TON_TAG>
```

> 💡 Remplace `<TON_TAG>` par le dernier tag stable, par exemple `2.19.1`.

---

## 🗂️ 2. Créer et configurer le Namespace

```bash
export NAMESPACE=awx
kubectl create ns ${NAMESPACE}
kubectl config set-context --current --namespace=$NAMESPACE
```

---

## ⚙️ 3. Déployer l'AWX Operator

```bash
make deploy NAMESPACE=awx
```

---

## 💾 4. Créer le PersistentVolumeClaim (PVC)

Ce volume persistant sera utilisé pour stocker les projets AWX.

```bash
tee awx-pvc.yml <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: awx-projects-claim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 2Gi
  storageClassName: local-path
EOF
```

Appliquer le PVC :

```bash
kubectl apply -f awx-pvc.yml
```

---

## 📄 5. Créer la déclaration du serveur AWX

> ⚠️ **Remplace `<DOMAINE_A_METTRE>`** par ton domaine réel (ex : `awx.mondomaine.local`). Si besoin d'un serveur DNS vous pouvez utilisé technitium DNS sur docker.

```bash
tee awx-deployment.yml <<EOF
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  postgres_storage_class: local-path
  postgres_storage_requirements:
    requests:
      storage: 8Gi
  projects_persistence: true
  garbage_collect_secrets: false
  projects_existing_claim: awx-projects-claim
  postgres_init_container_resource_requirements: {}
  postgres_resource_requirements: {}
  web_resource_requirements: {}
  task_resource_requirements: {}
  ee_resource_requirements: {}
  service_type: ClusterIP
  ingress_type: ingress
  ingress_class_name: traefik
  hostname: <DOMAINE_A_METTRE>
EOF
```

Appliquer la déclaration :

```bash
kubectl apply -f awx-deployment.yml
```

---

## 🔍 6. Vérifier le déploiement

### Vérifier l'état des Pods

```bash
kubectl get pods -l "app.kubernetes.io/managed-by=awx-operator"
```

Attends que tous les pods soient en statut `Running` avant de continuer. Cela peut prendre quelques minutes.

### Vérifier les Services

```bash
kubectl get svc -l "app.kubernetes.io/managed-by=awx-operator"
```

---

## 🔑 7. Récupérer le mot de passe administrateur

```bash
kubectl get secret awx-admin-password -o jsonpath="{.data.password}" | base64 --decode ; echo
```

> Le nom d'utilisateur par défaut est **`admin`**.

---

## 🌐 8. Accéder à l'interface web

Ouvre ton navigateur et accède à :

```
http://<DOMAINE_MIS_DANS_LE_HOSTNAME>
```

Connecte-toi avec :
- **Login** : `admin`
- **Mot de passe** : *(récupéré à l'étape précédente)*

---

## 🛠️ Dépannage

| Problème | Commande utile |
|---|---|
| Voir les logs de l'operator | `kubectl logs -l "app.kubernetes.io/name=awx-operator-controller-manager" -c awx-manager` |
| Voir les logs du pod AWX | `kubectl logs -l "app.kubernetes.io/name=awx" -c awx-web` |
| Voir tous les événements du namespace | `kubectl get events --sort-by='.lastTimestamp'` |
| Supprimer et recommencer | `kubectl delete -f awx-deployment.yml && kubectl delete -f awx-pvc.yml` |

---

## 📚 Ressources

- [AWX Operator — GitHub officiel](https://github.com/ansible/awx-operator)
- [Documentation K3s](https://docs.k3s.io/)
- [Documentation Ansible AWX](https://ansible.readthedocs.io/projects/awx/en/latest/)

---

*Guide rédigé pour un déploiement K3s avec Traefik Ingress et stockage `local-path`.*
