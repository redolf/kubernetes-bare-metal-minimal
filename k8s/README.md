Установка кластера kubernetes на Ubuntu
=========

Минимальные требования к виртуальным машинам
------------
- 3 master, 2 CPUs and 2 GBs RAM
- 2 workers, 2 CPUs and 2 GBs RAM

Действия для настройки кластера
--------------
- Отредактируйте inventory/inventory.yaml и vars/main.yaml
- Запустите ansible playbook и дождитесь его исполнения:

  ```
  ansible-playbook 1-k8s-prepare.yml
  ```
- Зайдите на одну из master-нод и выполните инициализацию кластера kubernetes:

  ```
  sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint k8s-master.poletaevlev.ru --service-dns-domain k8s.poletaevlev.ru
  kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --control-plane-endpoint "192.168.88.20:8888" \
  --upload-certs

  ```
  > Здесь мы указываем сеть для подов 10.244.0.0/16, она уже по умолчанию прописана в манифесте сетевого плагина Calico или Flannel который мы установим с помощью Helm. Если решите указать другую сеть, придётся поправить её в манифесте сетевого плагина.

- Подключите к кластеру kubernetes worker-ноды выполнив на них команду "kubeadm join" с параметрами полученными в консоли master-ноды при инициализации кластера
- Скопируйте конфигурацию подключения к кластеру на master ноде:

  ```
  mkdir -p $HOME/.kube && sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config && sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```
- Скопируйте конфигурацию подключения к кластеру на своё рабочее место:

  ```
  mkdir -p $HOME/.kube && scp redolf@k8s-master-1.poletaevlev.ru:$HOME/.kube/config $HOME/.kube/config
  ```
- Установите CNI (Container Network Interface) Calico на master-ноде:

  ```
  kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
  ```
- Проверьте что все ноды кластера имеют статус Ready с помощью команды:

  ```
  kubectl get no -o wide
  ```
- Проверьте что все системные поды кластера имеют статус Ready и счетчик Restart = 0 с помощью команды:

  ```
  kubectl get po -n kube-system
  ```
