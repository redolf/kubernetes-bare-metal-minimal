# Установка и настройка контроллера Ingress-Nginx в Kubernetes bare-metal
## Настройка контроллера Ingress-Nginx

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
  > hostNetwork=true, hostPort/enabled=true

- Установим ingress-nginx со значениями из файла values.yaml в namespace ingress-nginx
  ```
  helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace --values values.yaml
  ```
## Проверка работы контроллера Ingress-Nginx
- Проверим что ingress контроллер установился и сервис с типом LoadBalancer получил EXTERNAL-IP
  ```
  kubectl -n ingress-nginx get all
  ```
- Проверим что ingress контроллер работает задеплоив приложение с Nginx для теста из файла site-sample.yaml
  ```
  kubectl apply -f site-sample.yaml
  ```
  > После успешного деплоя добавляем dns запись для домена что описан в манифесте Ingress!
  >
  > - Перейдите по адресу что указан в манифесте Ingress, в данном случае http://t1.k8s.int в браузере. Если всё хорошо, вы увидите страницу по умолчанию nginx, на которой он поприветствует вас: **Welcome to nginx!**. 
  > - Чтобы протестировать доступность симулякра frontend Web API, в браузере пытаемся посмотреть страницу по адресу http://t1.k8s.int/test. Ответом должен быть **JSON**: 
  > ```
  > {"color":"yellow","message":"This is a Test","notify":"false",> "message_format":"text"}
  > ```
