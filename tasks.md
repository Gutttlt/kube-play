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

 kubectl apply -f https://raw.githubusercontent.com/Gutttlt/kube-play/main/blue-green-deploy-v1.yaml
deployment.apps/app-v1 created
$ kubectl apply -f https://raw.githubusercontent.com/Gutttlt/kube-play/main/blue-green-deploy-v2.yaml
deployment.apps/app-v2 created
$ kubectl get rs,pods
NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/app-v1-5d5dfcc7b    2         2         0       10s
replicaset.apps/app-v2-7c97464cdf   2         2         0       6s

NAME                          READY   STATUS              RESTARTS   AGE
pod/app-v1-5d5dfcc7b-88dnf    0/1     ContainerCreating   0          10s
pod/app-v1-5d5dfcc7b-rlpwh    0/1     ContainerCreating   0          10s
pod/app-v2-7c97464cdf-lvck4   0/1     ContainerCreating   0          6s
pod/app-v2-7c97464cdf-rbp8b   0/1     ContainerCreating   0          6s


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

$ kubectl apply -f https://raw.githubusercontent.com/Gutttlt/kube-play/main/blue-green-deploy-service.yaml
service/service created
$ kubectl get services
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   53s
service      ClusterIP   10.96.163.180   <none>        80/TCP    6s
$ curl -sD - http://10.96.163.180 | grep Apache
Server: Apache/2.4.41 (Unix)


$ curl -s  https://raw.githubusercontent.com/Gutttlt/kube-play/main/blue-green-deploy-service.yaml | sed -e 's/app-v1/app-v2/' | kubectl apply -f -
service/service configured


$ curl -sD - http://10.96.163.180 | grep Apache
Server: Apache/2.4.46 (Unix)


$ kubectl get pods --namespace=kube-system | grep ingress
nginx-ingress-controller-6fc5bcc8c9-czkwf   0/1     Running   0          28s


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
        
$ kubectl apply -f  https://raw.githubusercontent.com/Gutttlt/kube-play/main/blue-green-deploy-ingress.yaml
ingress.networking.k8s.io/app-ingress created

$ curl http://172.17.0.30/app 
<html><body><h1>It works!</h1></body></html>

