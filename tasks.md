**Загадка**

Демон для systemd

Напишите простой демон для systemd, который будет поддерживать работу процесса и перезапускаться в случае выхода из строя процесса.

<details>
  <summary>Отгадка</summary>
  Будем делать всё очень минималистично, но так, чтобы нескучно. Для минимализма сервисом будет netcat, пишущий в локальный файл:
  
  ```bash
  netcat -4 -l 3333 >> /tmp/dump
  ```
  А для веселья будем проверять non-privileged services, которые завезли в SystemD 239. Нужно же когда-нибудь это попробовать.
  Создадим директорию и юнит-файл:
  ```bash
onboard@dceu0858:~$ mkdir -p .config/systemd/user/
onboard@dceu0858:~$ cat >.config/systemd/user/mytest.service
[Unit]
Description="A test service"

[Service]
ExecStart=/bin/sh -c '/usr/bin/netcat -4 -l 3333 >> /tmp/dump'
Type=simple
Restart=always
```
Небольшие пояснения. Сознательно опущены After, Requres и прочее. /bin/sh вызывается для того, чтобы наш редирект в файл работал. По умолчанию SystemD не запускает никакого командного интерпретатора, а просто передаёт всё, что после имени бинарника, в качестве параметров. Type=simple потому, что sh умрёт вслед за netcat'ом, поскольку ему будет больше нечего делать. Ну, и Restart=always будет перезапускать сервис всегда, даже если exit code == 0.

Скажем, что systemd-userd для моего пользователя должен стартовать вместе с системой, иначе сервис умрёт при выходе пользователя из системы:
```bash
onboard@dceu0858:~$ sudo loginctl enable-linger onboard
```
Загрузим новые юниты:
```bash
onboard@dceu0858:~$ systemctl --user daemon-reload
```
Запустим, проверим статус:
```bash
onboard@dceu0858:~$ systemctl --user start mytest
onboard@dceu0858:~$ systemctl --user status mytest
● mytest.service - "A test service"
     Loaded: loaded (/home/onboard/.config/systemd/user/mytest.service; static; vendor preset: enabled)
     Active: active (running) since Thu 2021-04-15 10:21:57 CEST; 4s ago
   Main PID: 24886 (sh)
     CGroup: /user.slice/user-1000.slice/user@1000.service/mytest.service
             ├─24886 /bin/sh -c /usr/bin/netcat -4 -l 3333 >> /tmp/dump
             └─24887 /usr/bin/netcat -4 -l 3333

Apr 15 10:21:57 dceu0858 systemd[24709]: Started "A test service".
```

Убъём процесс и посмотрим, перезапустился ли он:
```bash
onboard@dceu0858:~$ kill 24887
onboard@dceu0858:~$ systemctl --user status mytest
● mytest.service - "A test service"
     Loaded: loaded (/home/onboard/.config/systemd/user/mytest.service; static; vendor preset: enabled)
     Active: active (running) since Thu 2021-04-15 10:22:27 CEST; 2s ago
   Main PID: 24890 (sh)
     CGroup: /user.slice/user-1000.slice/user@1000.service/mytest.service
             ├─24890 /bin/sh -c /usr/bin/netcat -4 -l 3333 >> /tmp/dump
             └─24891 /usr/bin/netcat -4 -l 3333

Apr 15 10:22:27 dceu0858 systemd[24709]: mytest.service: Scheduled restart job, restart counter is at 1.
Apr 15 10:22:27 dceu0858 systemd[24709]: Stopped "A test service".
Apr 15 10:22:27 dceu0858 systemd[24709]: Started "A test service".
```
Всё работает ровно как и заказано.
</details>


**Загадка**

Стратегии деплоймента

Сделайте реализацию blue/green стратегии деплоймента для Kubernetes на основе деплойментов, сервиса и ingress’а и опишите как переключать версии.


<details>
  <summary>Отгадка</summary>

В качестве примера приложений возьмём просто Apache двух разных версий.
Репликасет с Apache 2.4.41:
```bash
$ curl https://raw.githubusercontent.com/Gutttlt/kube-play/main/blue-green-deploy-v1.yaml
# V1: httpd 2.4.41
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
spec:
  selector:
    matchLabels:
      app: app-v1
  replicas: 2
  template:
    metadata:
      labels:
        app: app-v1
    spec:
      containers:
        - name: app-v1
          image: docker.io/library/httpd:2.4.41
          ports:
            - containerPort: 80
```

То же, но с Apache 2.4.46:
```bash
$ curl https://raw.githubusercontent.com/Gutttlt/kube-play/main/blue-green-deploy-v2.yaml
# V2: httpd 2.4.46
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v2
spec:
  selector:
    matchLabels:
      app: app-v2
  replicas: 2
  template:
    metadata:
      labels:
        app: app-v2
    spec:
      containers:
        - name: app-v2
          image: docker.io/library/httpd:2.4.46
          ports:
            - containerPort: 80
```

Применим оба:
```bash
$ kubectl apply -f https://raw.githubusercontent.com/Gutttlt/kube-play/main/blue-green-deploy-v1.yaml
deployment.apps/app-v1 created
$ kubectl apply -f https://raw.githubusercontent.com/Gutttlt/kube-play/main/blue-green-deploy-v2.yaml
deployment.apps/app-v2 created
```

Убедимся, что всё поднялось (ну, или ещё поднимается, слишком поздно заметил):
```bash
$ kubectl get rs,pods
NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/app-v1-5d5dfcc7b    2         2         0       10s
replicaset.apps/app-v2-7c97464cdf   2         2         0       6s

NAME                          READY   STATUS              RESTARTS   AGE
pod/app-v1-5d5dfcc7b-88dnf    0/1     ContainerCreating   0          10s
pod/app-v1-5d5dfcc7b-rlpwh    0/1     ContainerCreating   0          10s
pod/app-v2-7c97464cdf-lvck4   0/1     ContainerCreating   0          6s
pod/app-v2-7c97464cdf-rbp8b   0/1     ContainerCreating   0          6s
```

Далее можно пойти двумя путями: сделать сервис, который будем переключать между репликасетами, или же несколько сервисов, и переключать между ними будем на уровне трафик-менеджера. Из текста задания неясно, каким именно способом это должно быть реализовано, поэтому выбираем любой разумный. В данном случае будем переключать в сервисе (хотя вариант с переключением в ингрессе почему-то кажется более правильным).

Оределим сервис, посылающий на первую версию приложения:
```bash
$ curl https://raw.githubusercontent.com/Gutttlt/kube-play/main/blue-green-deploy-service.yaml
# A service
apiVersion: v1
kind: Service
metadata:
  name: service
spec:
  selector:
    app: app-v1
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Применим и убедимся, что он живой:
```bash
$ kubectl apply -f https://raw.githubusercontent.com/Gutttlt/kube-play/main/blue-green-deploy-service.yaml
service/service created
$ kubectl get services
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   53s
service      ClusterIP   10.96.163.180   <none>        80/TCP    6s
```
Теперь посмотрим, куда же он нас в действительности посылает:
```bash
$ curl -sD - http://10.96.163.180 | grep Apache
Server: Apache/2.4.41 (Unix)
```
Отлично, а теперь поменяем версию приложения на v2 (которая с Apache 2.4.46) и применим изменения:
```bash
$ curl -s  https://raw.githubusercontent.com/Gutttlt/kube-play/main/blue-green-deploy-service.yaml | sed -e 's/app-v1/app-v2/' | kubectl apply -f -
service/service configured
```
Куда нас теперь посылают?
```bash
$ curl -sD - http://10.96.163.180 | grep Apache
Server: Apache/2.4.46 (Unix)
```
Именно, в 2.4.46, как мы и хотели. Старые поды при этом живут, поскольку мы не просили их убивать. Потом можно убрать с помощью kubectl delete -f ...

Теперь убедимся, что у нас крутится какой-нибудь ингресс-контроллер:
```bash
$ kubectl get pods --namespace=kube-system | grep ingress
nginx-ingress-controller-6fc5bcc8c9-czkwf   0/1     Running   0          28s
```
Определим, что хотим отправить /app на наш сервис:
```bash
$ curl  https://raw.githubusercontent.com/Gutttlt/kube-play/main/blue-green-deploy-ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: app-ingress
  annotations: 
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - http:
        paths:
        - path: /app
          backend:
            serviceName: service
            servicePort: 80
```

Применим, насладимся:
```bash
$ kubectl apply -f  https://raw.githubusercontent.com/Gutttlt/kube-play/main/blue-green-deploy-ingress.yaml
ingress.networking.k8s.io/app-ingress created

$ kubectl get ingress
NAME          HOSTS   ADDRESS       PORTS   AGE
app-ingress   *       172.17.0.30   80      17m

$ curl http://172.17.0.30/app 
<html><body><h1>It works!</h1></body></html>
```

</details>
