Настройка отказоустойчивого кластера Kubernetes на Debian/Ubuntu
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
  kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --control-plane-endpoint "api-k8s.poletaev.local:8888" \
  --upload-certs

  ```
  > Здесь мы указываем сеть для подов 10.244.0.0/16, она уже по умолчанию прописана в манифесте сетевого плагина Calico или Flannel который мы установим с помощью Helm. Если решите указать другую сеть, придётся поправить её в манифесте сетевого плагина.

- Подключите к кластеру kubernetes master-ноды и worker-ноды выполнив на них команду "kubeadm join" с параметрами полученными в консоли master-ноды при инициализации кластера, для master и worker будет сгенерирован свой "kubeadm join".
- Скопируйте конфигурацию подключения к кластеру на каждой master ноде:

  ```
  mkdir -p $HOME/.kube && sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config && sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```
- Скопируйте конфигурацию подключения к кластеру на своё рабочее место:

  ```
  mkdir -p $HOME/.kube && scp $USER@control1-k8s.poletaev.local:$HOME/.kube/config $HOME/.kube/config
  ```
- Установите CNI (Container Network Interface) Calico на master-ноде:

  ```
  kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.5/manifests/calico.yaml
  ```
- Проверьте что все ноды кластера имеют статус Ready с помощью команды:

  ```
  kubectl get no -o wide
  ```
- Проверьте что все системные поды кластера имеют статус Ready и счетчик Restart не увиличивается с помощью команды:

  ```
  kubectl get po -n kube-system
  ```
