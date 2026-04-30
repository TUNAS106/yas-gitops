# YAS GitOps for ArgoCD

Thu muc nay duoc tao de tach thanh mot GitOps repository rieng cho phan nang cao ArgoCD. Thu muc `k8s/` cu cua source repo van duoc giu nguyen va co the tiep tuc deploy bang script Jenkins/Helm nhu truoc.

## Kien truc hien tai tu `k8s/`

Source repo cu deploy theo 2 nhom:

- `k8s/deploy`: cai dependency va ha tang dung chung.
- `k8s/charts`: Helm chart cho cac YAS application.

Namespace dang duoc dung trong cach deploy cu:

- `yas`: tat ca application YAS trong script `deploy-yas-applications.sh`.
- `postgres`: postgres operator, PostgreSQL, pgAdmin.
- `kafka`: Strimzi Kafka, Kafka cluster, AKHQ, Debezium connector.
- `elasticsearch`: ECK operator, Elasticsearch, Kibana.
- `keycloak`: Keycloak operator va Keycloak realm.
- `redis`: Redis.
- `zookeeper`: Zookeeper rieng.
- `observability`: da co chart/values nhung dang bi disable trong script hien tai.

Cach deploy application cu:

- `deploy-yas-configuration.sh` deploy chart `yas-configuration`.
- `deploy-yas-applications.sh` deploy cac chart application vao namespace `yas`.
- `deploy-developer-build.sh` deploy moi truong preview/developer bang Helm, co the override image tag cua mot service va expose service do bang `NodePort`.

Image/tag trong chart cu:

- Backend/BFF phan lon dung repository `tunas106/yas-*` voi tag mac dinh `latest`.
- UI dang dung `ghcr.io/nashtech-garage/yas-backoffice` va `ghcr.io/nashtech-garage/yas-storefront`.
- Chart dung `backend.image.tag` cho backend service, `ui.image.tag` cho UI service.

Expose service:

- Mac dinh service la `ClusterIP`.
- Script application cu expose qua Ingress host nhu `api.yas.local.com`, `backoffice.yas.local.com`, `storefront.yas.local.com`.
- Jenkins developer preview co the doi service can test sang `NodePort`.

## Cau truc GitOps

```text
gitops/
  argocd/
    applications/
      yas-dev.yaml
      yas-staging.yaml
  charts/
    ...
  environments/
    dev/
      apps/
        namespace.yaml
        applications.yaml
    staging/
      apps/
        namespace.yaml
        applications.yaml
  README.md
```

`charts/` la ban copy rieng tu `k8s/charts`. Cach nay phu hop khi tach `gitops/` thanh repo GitOps moi, vi ArgoCD se chi can clone GitOps repo va khong can truy cap source repo chinh.

Khong nen de Application trong GitOps repo tham chieu `../k8s/charts` cua source repo cu, vi khi push sang repo moi duong dan do se khong con ton tai.

## Moi truong

`dev`:

- Namespace: `yas-dev`.
- ArgoCD root application: `yas-dev`.
- Host demo:
  - `api-dev.yas.local.com`
  - `backoffice-dev.yas.local.com`
  - `storefront-dev.yas.local.com`
- Image tag ban dau: `main`.
- CI nen thay `main` bang commit SHA moi nhat sau khi build va push image.

`staging`:

- Namespace: `yas-staging`.
- ArgoCD root application: `yas-staging`.
- Host demo:
  - `api-staging.yas.local.com`
  - `backoffice-staging.yas.local.com`
  - `storefront-staging.yas.local.com`
- Image tag ban dau: `v1.0.0`.
- CI release nen thay bang release tag that, vi du `v1.2.3`.

## Push sang GitOps repo moi

1. Tao GitHub repo moi, vi du `yas-gitops`.
2. Copy noi dung thu muc `gitops/` nay vao repo do.
3. Commit va push:

```bash
git init
git add .
git commit -m "Initial YAS GitOps structure"
git branch -M main
git remote add origin https://github.com/<your-user>/yas-gitops.git
git push -u origin main
```

4. Doi tat ca `repoURL` trong cac file YAML tu:

```text
https://github.com/TUNAS106/yas-gitops.git
```

thanh URL repo GitOps that cua ban.

## Apply ArgoCD Application dau tien

Sau khi cai ArgoCD trong namespace `argocd`, apply root application:

```bash
kubectl apply -f argocd/applications/yas-dev.yaml
kubectl apply -f argocd/applications/yas-staging.yaml
```

Root application se tao cac Application con trong:

```text
environments/dev/apps/
environments/staging/apps/
```

Moi Application con se sync mot Helm chart vao namespace tuong ung.

## CI cap nhat image tag

ArgoCD theo doi Git, khong tu biet Docker Hub co image moi. Sau khi CI build va push image, CI phai commit thay doi image tag vao GitOps repo.

Vi du dev sau khi push `main`:

```bash
yq -i '
  (.spec.source.helm.parameters[] | select(.name == "backend.image.tag").value) = "a1b2c3d"
' environments/dev/apps/applications.yaml

git add environments/dev/apps/applications.yaml
git commit -m "Deploy dev image a1b2c3d"
git push
```

Vi du staging khi tao release tag:

```bash
yq -i '
  (.spec.source.helm.parameters[] | select(.name == "backend.image.tag").value) = "v1.2.3"
' environments/staging/apps/applications.yaml

git add environments/staging/apps/applications.yaml
git commit -m "Deploy staging v1.2.3"
git push
```

Neu chi muon update mot service, nen loc them theo Application name. Vi du update `tax` o dev:

```bash
yq -i '
  (. | select(.metadata.name == "yas-dev-tax").spec.source.helm.parameters[]
     | select(.name == "backend.image.tag")).value = "a1b2c3d"
' environments/dev/apps/applications.yaml
```

Voi UI service thi doi parameter `ui.image.tag` thay vi `backend.image.tag`.

## Rollback bang Git

Rollback chi can revert commit GitOps:

```bash
git log --oneline
git revert <commit-can-rollback>
git push
```

ArgoCD se thay GitOps repo thay doi va sync lai image tag cu.

## Kiem tra ket qua

Kiem tra namespace:

```bash
kubectl get ns | grep yas
```

Kiem tra pod/service/ingress:

```bash
kubectl get pod -n yas-dev
kubectl get svc -n yas-dev
kubectl get ingress -n yas-dev

kubectl get pod -n yas-staging
kubectl get svc -n yas-staging
kubectl get ingress -n yas-staging
```

Kiem tra Application trong ArgoCD:

```bash
kubectl get applications -n argocd
```

Trong ArgoCD UI, ban se thay:

- `yas-dev`
- `yas-staging`
- cac app con nhu `yas-dev-tax`, `yas-staging-tax`, `yas-dev-storefront-bff`, ...

## Luu y khi demo

- Jenkins `developer_build` van nen chi deploy preview, khong deploy vao `yas-dev` hoac `yas-staging`.
- ArgoCD quan ly `yas-dev` va `yas-staging`.
- Ha tang nang nhu PostgreSQL, Kafka, Elasticsearch, Keycloak, Redis van dung chung tu cac namespace ha tang cu.
- Neu Minikube thieu tai nguyen, co the tam thoi xoa bot Application con khong can demo trong `environments/*/apps/applications.yaml`, vi du `payment-paypal`, `recommendation`, `search`.
- Cac host `*-dev.yas.local.com` va `*-staging.yas.local.com` can duoc them vao file hosts tro ve IP Minikube.
- Neu login Keycloak bi loi redirect URL, can them cac URL dev/staging vao Keycloak client redirect URI.
