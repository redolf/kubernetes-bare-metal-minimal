# Настройка отказоустойчивого кластера Kubernetes на Debian/Ubuntu/Rocky Linux


## Минимальные требования к виртуальным машинам
- 3 master, 2 CPUs and 2 GBs RAM
- 2 workers, 2 CPUs and 2 GBs RAM

## Действия для настройки кластера
- Отредактируйте inventory/inventory.yaml и vars/main.yaml
- Запустите ansible playbook, введите запрашиваемые пререквизиты (во время работы возможно придётся отлаживать и немного подпиливать ansible роль, это нормально):

```bash
ansible-playbook 1-k8s-prepare.yml
```

- Зайдите на одну из master-нод и выполните инициализацию кластера kubernetes:

```bash
kubeadm init \
--pod-network-cidr=10.244.0.0/16 \
--control-plane-endpoint "k8s-api.pla.int:8888" \
--upload-certs
```

  > pod-network-cidr - сеть для подов
  > control-plane-endpoint - доменное имя или ip Api Kubernetes

- Подключите к кластеру kubernetes master-ноды и worker-ноды выполнив на них команду "kubeadm join" с параметрами полученными в консоли master-ноды при инициализации кластера, для master и worker будет сгенерирован свой "kubeadm join".

- Скопируйте конфигурацию подключения к кластеру на каждой master ноде:

```bash
mkdir -p $HOME/.kube && sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config && sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- Скопируйте конфигурацию подключения к кластеру на своё рабочее место:

```bash
mkdir -p $HOME/.kube && scp $USER@k8s-control1.pla.int:$HOME/.kube/config $HOME/.kube/config
```

Кластер установлен но ноды кластера не будут Ready

```bash
kubectl get no -o wide
```

Далее нам нужно нужно определиться какой CNI (Container Network Interface) плагин будем использовать и установить его чтобы работала маршрутизация и сеть подов между нод кластера

[Выбор и установка CNI плагина](../cni)