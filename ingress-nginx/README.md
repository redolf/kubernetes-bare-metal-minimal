# Установка и настройка контроллера Ingress-Nginx в Kubernetes bare-metal

- Добавим репозиторий ingress-nginx и обновим список в кластере kubernetes

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx && \
helm repo update
```

- Установим Ingress-Nginx в namespace `ingress-nginx`
    >controller.service.type=LoadBalancer - указывает что сервис Ingress-Nginx у нас будет с VIP

```bash
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

- Проверим что ingress контроллер установился и сервис с типом LoadBalancer получил EXTERNAL-IP

```bash
kubectl -n ingress-nginx get all
```

- Создадим Ingress для Hubble Cilium (если используется CNI Cilium с включенным Hubble)
    >При деплое Cilium если мы указали в конфигурации для Hubble тип сервиса `LoadBalance` и сервису был присвоен VIP (Virtual IP) из заданного пула.   
    >Сейчас мы можем использовать Ingress Controller, сервис которого так же `LoadBalance` и разруливать трафик по приложениям внутри кластера по приложениям имея всего 1 VIP, а сервису hubble-ui сменить тип на ClusterIP чтобы освободить 1 VIP

```yaml
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hubble-ingress
  namespace: kube-system
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: hubble.pla.int
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hubble-ui
            port:
              number: 80
EOF
```

- Вариант проверки работы через деплой тест-приложения с Ingress

```yaml
INGRESS_HOST=t1.pla.int
NAMESPACE_TEST=test-ingress-app

kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: ${NAMESPACE_TEST}
  labels:
    app.kubernetes.io/name: test-ingress-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spa-deployment
  namespace: ${NAMESPACE_TEST}
  labels:
    app.kubernetes.io/name: spa
    app.kubernetes.io/component: spa
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: spa
      app.kubernetes.io/component: spa
  template:
    metadata:
      labels:
        app.kubernetes.io/name: spa
        app.kubernetes.io/component: spa
    spec:
      containers:
      - name: spa
        image: nginx
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                - spa
              - key: app.kubernetes.io/component
                operator: In
                values:
                - spa
            topologyKey: "kubernetes.io/hostname"
---
apiVersion: v1
kind: Service
metadata:
  name: spa
  namespace: ${NAMESPACE_TEST}
  labels:
    app.kubernetes.io/component: spa
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: spa
    app.kubernetes.io/component: spa
  ports:
  - port: 80
    targetPort: 80
    name: http
    protocol: TCP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-api
  namespace: ${NAMESPACE_TEST}
  labels:
    app.kubernetes.io/name: frontend-api
    app.kubernetes.io/component: frontend-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: frontend-api
      app.kubernetes.io/component: frontend-api
  template:
    metadata:
      labels:
        app.kubernetes.io/name: frontend-api
        app.kubernetes.io/component: frontend-api
    spec:
      containers:
      - name: frontend-api
        image: chentex/go-rest-api
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                - frontend-api
              - key: app.kubernetes.io/component
                operator: In
                values:
                - frontend-api
            topologyKey: "kubernetes.io/hostname"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-api
  namespace: ${NAMESPACE_TEST}
  labels:
    app.kubernetes.io/name: frontend-api
    app.kubernetes.io/component: frontend-api
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: frontend-api
    app.kubernetes.io/component: frontend-api
  ports:
  - port: 80
    targetPort: 8080
    name: http
    protocol: TCP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-test-site
  namespace: ${NAMESPACE_TEST}
spec:
  ingressClassName: nginx
  rules:
  - host: ${INGRESS_HOST}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: spa
            port:
              number: 80
      - path: /test
        pathType: Prefix
        backend:
          service:
            name: frontend-api
            port:
              number: 80
EOF
```

- После успешного деплоя добавляем dns запись для домена что описан в манифесте Ingress с VIP который использует сервис Ingress-Nginx, посмотреть можно например в задеплоенном манифесте Ingress в поле Address

```bash
kubectl -n test-ingress-app get ingress
```

- Перейдите по адресу что указан в манифесте Ingress, в данном случае http://t1.pla.int в браузере. Если всё хорошо, вы увидите страницу по умолчанию nginx, на которой он поприветствует вас: **Welcome to nginx!**.
- Чтобы протестировать доступность симулякра frontend Web API, в браузере пытаемся посмотреть страницу по адресу http://t1.pla.int/test. Ответом должен быть **JSON**:

```json
{"color":"yellow","message":"This is a Test","notify":"false",> "message_format":"text"}
```