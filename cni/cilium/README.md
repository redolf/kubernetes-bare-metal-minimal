## Cilium

Установим клиент Cilium на машину администратора или Control node, смотря откуда удобно подключаться

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

Чтобы получить доступ к данным о наблюдаемости, собранным Hubble,  установим клиент Hubble CLI на машину администратора или Control node

```bash
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/main/stable.txt)
HUBBLE_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}

```
### Cilium с заменой kube-proxy и MetalLB

Cilium в таком режиме полностью берёт на себя ClusterIP/NodePort/ExternalIP/LoadBalancer/HostPort
#### Отключение kube-proxy (*если kube-proxy был установлен*)

- Удалить kube‑proxy DaemonSet и ConfigMap
>Удаление ConfigMap нужно, чтобы kubeadm при апгрейдах не поставил kube‑proxy заново

```bash
kubectl -n kube-system delete ds kube-proxy && \
kubectl -n kube-system delete cm kube-proxy
```

- Почистить правила iptables от KUBE‑цепочек на КАЖДОМ узле
>Это убирает мусор от kube‑proxy, чтобы он не конфликтовал с eBPF‑балансировкой Cilium

```bash
iptables-save | grep -v KUBE | iptables-restore
```
​
#### Установка и конфигурация CNI плагина Cilium

- Установка Cilium
>API_SERVER_IP - наш ip или доменной имя API kubernetes
>API_SERVER_PORT - порт API kubernetes
>kubeProxyReplacement=true - замена обязанностей kube-proxy

```bash
API_SERVER_IP=k8s-api.pla.int
API_SERVER_PORT=8888

helm repo add cilium https://helm.cilium.io/ && \
helm repo update && \
helm upgrade --install cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=${API_SERVER_IP} \
  --set k8sServicePort=${API_SERVER_PORT} \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.ui.service.type=LoadBalancer \
  --set loadBalancer.mode=snat \
  --set loadBalancer.bpfSocketLBHostnsOnly=true \
  --set l2announcements.enabled=true
  # --set l2announcements.policy.default-l2-announcement.interfaces[0]=eth0 \
  # --set l2announcements.policy.default-l2-announcement.loadBalancerIPs=true
```

| Параметр                                                            | Описание                                                        |
| ------------------------------------------------------------------- | --------------------------------------------------------------- |
| reuse-values                                                        | Сохранить предыдущие настройки                                  |
| kubeProxyReplacement=true                                           | Заменить kube-proxy на eBPF (0% CPU, максимальная скорость)     |
| hubble.enabled=true                                                 | Включить Hubble (сбор метрик сети)                              |
| hubble.relay.enabled=true                                           | Hubble Relay (GRPC шлюз для UI)                                 |
| hubble.ui.enabled=true                                              | Hubble UI (веб-интерфейс)                                       |
| hubble.ui.service.type=LoadBalancer                                 | Hubble UI как LoadBalancer → L2 IP                              |
| socketLB.hostNamespaceOnly=true                                     | SocketLB только в host namespace (совместимость с L2)           |
| loadBalancer.mode=snat                                              | SNAT режим (совместим с VXLAN туннелем)                         |
| loadBalancer.bpfSocketLBHostnsOnly=true                             | BPF SocketLB в hostNS (фиксит ARP=0 проблемы)                   |
| l2announcements.enabled=true                                        | Включить L2 Announcements (ARP резолв для LB IP) **❌ баг Helm** |
| l2announcements.policy.default-l2-announcement.interfaces[0]=eth0   | Интерфейс для ARP (eth0 → MAC ответы!) **❌ баг Helm**           |
| l2announcements.policy.default-l2-announcement.loadBalancerIPs=true | Объявлять LoadBalancer IP **❌ баг Helm**                        |

- Проверка статуса Cilium

```bash
cilium status --wait
```

```bash
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium                   Desired: 3, Ready: 3/3, Available: 3/3
DaemonSet              cilium-envoy             Desired: 3, Ready: 3/3, Available: 3/3
Deployment             cilium-operator          Desired: 2, Ready: 2/2, Available: 2/2
Containers:            cilium                   Running: 3
                       cilium-envoy             Running: 3
                       cilium-operator          Running: 2
                       clustermesh-apiserver
                       hubble-relay
Cluster Pods:          3/3 managed by Cilium
Helm chart version:    1.19.1
Image versions         cilium             quay.io/cilium/cilium:v1.19.1@sha256:41f1f74a0000de8656f1de4088ea00c8f2d49d6edea579034c73c5fd5fe01792: 3
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.35.9-1770979049-232ed4a26881e4ab4f766f251f258ed424fff663@sha256:8188114a2768b5f49d6ce58e168b20d765e0fbc64eee0d83241aa2b150ccd788: 3
                       cilium-operator    quay.io/cilium/operator-generic:v1.19.1@sha256:e7278d763e448bf6c184b0682cf98cdca078d58a27e1b2f3c906792670aa211a: 2
```

- Создадим пул IP и Policy для выдачи ip сервисам LoadBalancer и доступа к ним
> STARP_IP - Начало пула IP
 >STOP_IP - Конец пула IP
 
```yaml
STARP_IP="192.168.100.200"
STOP_IP="192.168.100.207"

kubectl apply -f - <<'EOF'
apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool
metadata:
  name: default-pool
spec:
  blocks:
    - start: ${STARP_IP}
      stop: ${STOP_IP}
---
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: default-l2-announcement
  namespace: kube-system
spec:
  interfaces: ["eth0"]
  loadBalancerIPs: true
EOF
```