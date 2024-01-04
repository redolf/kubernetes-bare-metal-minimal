Установка и настройка балансировщика MetalLB для кластера Kubernetes bare-metal
=========

Действия для настройки балансировщика MetalLB
--------------
- Установим MetalLB
  
  ```
  kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
  ```
- Отредактируйте пул ip в metallb/ippool.yaml
- Установим манифесты IPAddressPool и L2Advertisement:

  ```
  kubectl apply -f ippool.yml
  ```
