Установка и настройка контроллера Ingress-Nginx в Kubernetes bare-metal
=========

Действия для настройки контроллера Ingress-Nginx
--------------
- Добавим репозиторий ingress-nginx и обновим список в кластере kubernetes
  
  ```
  helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx && helm repo update
  ```
- Создадим namespace для ingress-nginx
  
  ```
  kubectl create ns ingress-nginx
  ```
- Получить все параметры ingress-nginx в файл для дальнейшего конфигурирования
  ```
  helm show values ingress-nginx --repo https://kubernetes.github.io/ingress-nginx >> values.yaml
  ```
  > Были изменены параметры для работы контроллера на bare-metal:
  > hostNetwork=true, hostPort/enabled=true, kind=DaemonSet

- Установим ingress-nginx со значениями из файла values.yaml в namespace ingress-nginx
  ```
  helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace --values values.yaml
  ```
