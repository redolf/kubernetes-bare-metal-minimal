# Установка и настройка хранилища Longhorn
> Longhorn — это простая в использовании распределённая блочная система хранения данных. Обеспечивает высокодоступное постоянное хранилище для Kubernetes. Вы можете применять встроенные функции инкрементных снимков, резервного копирования и хранения резервных копии во вторичном хранилище, совместимом с NFS или S3. Тем самым обеспечивается безопасность данных в кластере Kubernetes.

## Требования к установке
- kubernetes — не ниже v1.21, Docker — не ниже v1.13, Containerd — не ниже v1.3.7;
- установлен пакет open-iscsi и запущена служба iscsid; 
- установлен клиент NFSv4, необходимый для функции резервного копирования, и ReadWriteMany (RWX);
- файловая система ext4 или XFS;
установлены пакеты bash, curl, findmnt, grep, awk, blkid, lsblk;
- в кластере включено распространение монтирования (mount propagation).

> ### Исходя из требований для предварительной настройки написана ansible-роль:
> - Устанавливает на ноды в группе **k8s_worker**:
>   - nfs-common
>   - open-iscsi
> - Включает и стартует демона **iscsid** на нодах в группе **k8s_worker**
> - Включает mount propagation в параметрах кластера внося запись в **/etc/kubernetes/manifests/kube-apiserver.yaml** на ноды в группе **k8s_master**

## Настройка кластера для установки Longhorn
- Запустить ansible роль
  ```
  ansible-playbook longhorn-k8s-prepare.yml
  ```
## Подготовка Longhorn и установка в кластер

- Установите репозиторий Longhorn
  ```
  helm repo add longhorn https://charts.longhorn.io
  helm repo update
  ```
- Сохраните values longhorn в файл values.yaml
  ```
  helm show values longhorn/longhorn > values.yaml
  ```
- Изменить параметры longhorn в values.yaml
  - Директория для хранения данных на хосте `defaultDataPath: /mnt/longhorn/data`
    
  - Планирование реплик на узлах с существующими исправными репликами того же тома `replicaSoftAntiAffinity: true`
  - Автоматическая балансировка реплик при обнаружении доступного узла `replicaAutoBalance: best-effort`
  - Включение ingress для Longhorn UI
    ```
    ingress:
        enabled: true
    ```
  - Домен по которому будет доступен Longhorn UI
    ```
    ingress:
        host: lh.example.com
    ```
  - Задаём Class для ingress
    ```
    ingress:
        ingressClassName: nginx
    ```
  - Задаём Annotations для ingress. Включаем basic-auth для авторизации в UI
    ```
    ingress:
        annotations:
            nginx.ingress.kubernetes.io/auth-type: basic
            nginx.ingress.kubernetes.io/ssl-redirect: 'false'
            nginx.ingress.kubernetes.io/auth-secret: basic-auth
            nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required '
            nginx.ingress.kubernetes.io/proxy-body-size: 10000m
    ```
- Подготовим secret для базовой авторизации в UI Longhorn
  ```
  USER='ЛОГИН'; PASSWORD='ПАРОЛЬ'; echo "${USER}:$(openssl passwd -stdin -apr1 <<< ${PASSWORD})" >> /tmp/auth

  kubectl create ns longhorn-system

  kubectl -n longhorn-system create secret generic basic-auth --from-file=/tmp/auth
  ```
- Устанавливаем longhorn в кластер
  ```
  helm upgrade --install longhorn longhorn/longhorn -f values.yaml -n longhorn-system
  ```

> ## Для разрешения удаления Longhron
> Установите значение true для флага deleting-confirmation-flag, выполнив команду: `kubectl -n longhorn-system patch -p '{"value": "true"}' --type=merge lhs deleting-confirmation-flag` после чего уже можно удалять чарт `helm uninstall longhorn -n longhorn-system`
