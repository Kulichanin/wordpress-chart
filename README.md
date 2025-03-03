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

```bash
mysql -h127.0.0.1 -u root -pPASS -e "CREATE DATABASE DB_NAME; CREATE USER "USER_APP"@"%" IDENTIFIED BY PASS_APP"; GRANT ALL PRIVILEGES ON NAME_DB.\* TO "USER_APP"@"%"; FLUSH PRIVILEGES; EXIT"
```

Создадим секрет для работу через `helm secret`

```bash
sops --encrypted-suffix private_env_variables --pgp key_fp wordpress/data.sec.yaml
```

Пример секрета

```yaml
public_env_variables:
  WORDPRESS_DEBUG: 1
private_env_variables:
  WORDPRESS_DB_HOST: "svc.cluster.local"
  WORDPRESS_DB_USER: "user"
  WORDPRESS_DB_PASSWORD: "user"
  WORDPRESS_DB_NAME: "user"
```
