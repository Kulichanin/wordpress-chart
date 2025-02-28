# Развертывание инфарструктуры

## NSF Provisioner

```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install --namespace nfs-provisioner --create-namespace nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=x.x.x.x \
    --set nfs.path=/exported/path 
```

## Database

Установка mysql оператора для работы с кластером `InnoDBCluster`

```bash
helm install mysql-operator mysql-operator/mysql-operator --namespace mysql-system --create-namespace
```

Cекрет для создания пользователя root mysql находится в `internal/database/mypwds.sec.yaml` и зашифрован sops

Манифест с кластером описан в `internal/database/deploy.yaml`

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

```bash
helm install nginx-logs ingress-nginx/ingress-nginx --create-namespace \
 --namespace ingress-logs --set controller.ingressClass="nginx-logs" \
 --set controller.ingressClassResource.name="nginx-logs" 
```

```bash
helm install nginx-prod ingress-nginx/ingress-nginx --create-namespace \
 --namespace ingress-prod --set controller.ingressClass="nginx-prod" \
 --set controller.ingressClassResource.name="nginx-prod" 
```

```bash
helm install nginx-dev ingress-nginx/ingress-nginx --create-namespace \
 --namespace ingress-dev --set controller.ingressClass="nginx-dev" \
 --set controller.ingressClassResource.name="nginx-dev" 
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

```bash
helm pull prometheus-community/prometheus-adapter --untar
```

Настроить в values.yaml ingress и установить через helm

```bash
helm install kube-prometheus-stack --namespace monitoring --create-namespace --wait -f kube-prometheus-stack/values.yaml ./kube-prometheus-stack
```
