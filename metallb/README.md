Установка и настройка балансировщика MetalLB для кластера Kubernetes bare-metal
=========

Действия для настройки балансировщика MetalLB
--------------
- Установим MetalLB
  
  ```
  kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
  ```
- Отредактируйте пул ip в metallb/ippool.yaml
- Установим манифесты IPAddressPool и L2Advertisement:

  ```
  kubectl apply -f ippool.yml
  ```


# Metallb

[https://metallb.universe.tf/](https://metallb.universe.tf/)

Если kubeproxy запущен без `strictARP: true`. Исправим это, отредактировав
соответствующий configmap:

```shell
kubectl edit configmap -n kube-system kube-proxy
```

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```

Установим metallb:

```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
```

Отредактируйте пул ip в metallb/ippool.yaml

Установим манифесты IPAddressPool и L2Advertisement:

  ```shell
  kubectl apply -f ippool.yml
  ```
