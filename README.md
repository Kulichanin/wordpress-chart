# Развертывание инфарструктуры

## NSF Provisioner

```bash
sudo apt install -y nfs-common nfs4-acl-tools 
```

Получение helm chart

```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm pull nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --untar
```

```bash
helm install --namespace nfs-provisioner --create-namespace nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=x.x.x.x \
    --set nfs.path=/opt/nfs 
```

## Cert-manager & nginx-ingress-controller

Установка нескольких nginx ingress controller. Это свяазно с тем что разные ingress контроллеры будут использоваться под разные задачи

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx --force-update
```

```bash
helm install nginx-metrics ingress-nginx/ingress-nginx --create-namespace \
 --namespace ingress-metrics --set controller.ingressClass="nginx-metrics" \
 --set controller.ingressClassResource.name="nginx-metrics" 
```

Установка Cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io --force-update
```

```bash
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.0 \
  --set crds.enabled=true
```

Создание менеджера сертификатов для nginx-ingress-controller описано в `internal/cert-manager & nginx-ingress-controller/deploy.yaml`

## Prometheus stack

Получение helm chart

```bash
helm pull prometheus-community/prometheus-adapter --untar
```

Настроить в values.yaml ingress и установить через helm

```bash
helm install kube-prometheus-stack --namespace monitoring --create-namespace --wait -f kube-prometheus-stack/values.yaml ./kube-prometheus-stack
```

## Database

Установка mysql оператора для работы с кластером `InnoDBCluster`

```bash
helm repo add mysql-operator https://mysql.github.io/mysql-operator/
helm repo update
```

```bash
helm install mysql-operator mysql-operator/mysql-operator --namespace mysql-system --create-namespace
```

Cекрет для создания пользователя root mysql находится в `internal/database/mypwds.sec.yaml` и зашифрован sops

Манифест с кластером описан в `internal/database/deploy.yaml`

## EFK stack

Для развертывания elastik + kibana используется eck-operator.

```bash
git clone https://github.com/elastic/cloud-on-k8s.git
```

```bash
helm install elastic-operator eck-operator -n elastic-system --create-namespace
```

В ek_deploy.yaml описан деплой elastik + kibana

Для развертывания fluent-bit необходимо скачать репозиторий c помощью helm

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
helm pull fluent/fluent-bit --untar
```

Установка с корректной конфигурацией

```bash
helm install fluent-bit -n elastic-system ./fluent-bit/ -f fluent-bit/values.yaml
```

Получим пароль из соответствующего секрета

```bash
kubectl get secrets elastic-cluster-es-elastic-user -o json | jq -r .data.elastic | base64 -d
```

## Приложение Wordpress

Перед развертыванием приложения подготовим базу данных

```bash
kubectl port-forward service/mysql-cluster 3306 -n mysql-cluster
```

Создадим для приложения две базы

Одну для prod окружения

```bash
mysql -h127.0.0.1 -u root -pPASS -e "CREATE DATABASE DB_NAME_PROD; CREATE USER "USER_APP_PROD"@"%" IDENTIFIED BY PASS_APP_PROD"; GRANT ALL PRIVILEGES ON DB_NAME_PROD.\* TO "USER_APP_PROD"@"%"; FLUSH PRIVILEGES; EXIT"
```

Вторую для dev окружения

```bash
mysql -h127.0.0.1 -u root -pPASS -e "CREATE DATABASE DB_NAME_DEV; CREATE USER "USER_APP_DEV"@"%" IDENTIFIED BY PASS_APP_DEV"; GRANT ALL PRIVILEGES ON DB_NAME_DEV.\* TO "USER_APP_DEV"@"%"; FLUSH PRIVILEGES; EXIT"
```

Создадим секрет для работу через `helm secret`

```bash
sops --encrypted-suffix private_env_variables --pgp key_fp wordpress/secrets.prod.yaml
```

Пример секрета для dev

```yaml
public_env_variables:
  WORDPRESS_DEBUG: 1
private_env_variables:
  WORDPRESS_DB_HOST: "svc.cluster.local"
  WORDPRESS_DB_USER: "USER_APP_PROD"
  WORDPRESS_DB_PASSWORD: "PASS_APP_PROD"
  WORDPRESS_DB_NAME: "DB_NAME_PROD"
```

```bash
sops --encrypted-suffix private_env_variables --pgp key_fp wordpress/secrets.dev.yaml
```

Пример секрета для prod

```yaml
private_env_variables:
  WORDPRESS_DB_HOST: "svc.cluster.local"
  WORDPRESS_DB_USER: "USER_APP_DEV"
  WORDPRESS_DB_PASSWORD: "PASS_APP_DEV"
  WORDPRESS_DB_NAME: "DB_NAME_DEV"
```

Создадим два namespace для деплоя в него приложения

```bash
kubectl create ns dev prod 
```

Создадим ServiceAccount dev-wordpress, prod-wordpress создадим роли и привяжем их к dev-wordpress, prod-wordpress. Также создадим токен для авторизации в кластере k8s

<!-- !TODO Добавить роли для configmap, hpa, ingress, pvc -->

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dev-wordpress
  namespace: dev
automountServiceAccountToken: true  
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prod-wordpress
  namespace: prod
automountServiceAccountToken: true  
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-wordpress
  namespace: dev
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]    
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["monitoring.coreos.com"]
    resources: ["servicemonitors"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]       
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: prod-wordpress
  namespace: prod
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]    
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]  
  - apiGroups: ["monitoring.coreos.com"]
    resources: ["servicemonitors"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]     
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-rolebinding
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: dev-sa
    namespace: dev
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: prod-rolebinding
  namespace: prod
subjects:
  - kind: ServiceAccount
    name: prod-sa
    namespace: prod
roleRef:
  kind: Role
  name: prod-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: dev-sa-token
  namespace: dev
  annotations:
    kubernetes.io/service-account.name: dev-sa
---
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: prod-sa-token
  namespace: prod
  annotations:
    kubernetes.io/service-account.name: prod-sa    
```

Конфиг для переменной env gitlab ci/cd KUBECONFIG_DEV

```yaml
apiVersion: v1
kind: Config
clusters:
- name: kubernetes
  cluster:
    server: <API_SERVER> # Можно взять из `kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'`
    certificate-authority-data: <CA_CERT>  # Можно взять из `kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}'`
contexts:
- name: dev-sa-context
  context:
    cluster: kubernetes
    namespace: test
    user: dev-sa
current-context: dev-sa-context
users:
- name: dev-sa
  user:
    token: <TOKEN> #  Можно взять из `kubectl get secret dev-sa-token -n test -o jsonpath='{.data.token}' | base64 --decode`
```

Конфиг для переменной env gitlab ci/cd KUBECONFIG_PROD

```yaml
apiVersion: v1
kind: Config
clusters:
- name: kubernetes
  cluster:
    server: <API_SERVER> # Можно взять из `kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'`
    certificate-authority-data: <CA_CERT>  # Можно взять из `kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}'`
contexts:
- name: prod-sa-context
  context:
    cluster: kubernetes
    namespace: test
    user: prod-sa
current-context: prod-sa-context
users:
- name: prod-sa
  user:
    token: <TOKEN> #  Можно взять из `kubectl get secret prod-sa-token -n test -o jsonpath='{.data.token}' | base64 --decode`
```
