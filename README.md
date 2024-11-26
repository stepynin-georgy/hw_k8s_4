# Домашнее задание к занятию «Сетевое взаимодействие в K8S. Часть 1»

### Цель задания

В тестовой среде Kubernetes необходимо обеспечить доступ к приложению, установленному в предыдущем ДЗ и состоящему из двух контейнеров, по разным портам в разные контейнеры как внутри кластера, так и снаружи.

------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым Git-репозиторием.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Описание](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) Deployment и примеры манифестов.
2. [Описание](https://kubernetes.io/docs/concepts/services-networking/service/) Описание Service.
3. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

### Задание 1. Создать Deployment и обеспечить доступ к контейнерам приложения по разным портам из другого Pod внутри кластера

1. Создать Deployment приложения, состоящего из двух контейнеров (nginx и multitool), с количеством реплик 3 шт.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hw-4
spec:
  selector:
    matchLabels:
      app: hw-4
  replicas: 3
  template:
    metadata:
      labels:
        app: hw-4
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.4
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 8080
        env:
        - name: HTTP_PORT
          value: "8080"
```

```
user@k8s:/opt/hw_k8s_4$ microk8s kubectl apply -f task-1-deployment.yaml
deployment.apps/hw-4 created
user@k8s:/opt/hw_k8s_4$ microk8s kubectl get deployment
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
hw-4   3/3     3            3           77s
```

2. Создать Service, который обеспечит доступ внутри кластера до контейнеров приложения из п.1 по порту 9001 — nginx 80, по 9002 — multitool 8080.

```
apiVersion: v1
kind: Service
metadata:
  name: hw-4-svc
  namespace: default
spec:
  selector:
    app: hw-4
  ports:
    - protocol: TCP
      name: nginx
      port: 9001
      targetPort: 80
    - protocol: TCP
      name: multitool
      port: 9002
      targetPort: 8080
```

```
user@k8s:/opt/hw_k8s_4$ microk8s kubectl apply -f task-1-svc.yaml
service/hw-4-svc created
user@k8s:/opt/hw_k8s_4$ microk8s kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
hw-4-svc     ClusterIP   10.152.183.34   <none>        9001/TCP,9002/TCP   8s
kubernetes   ClusterIP   10.152.183.1    <none>        443/TCP             6d23h
```

3. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложения из п.1 по разным портам в разные контейнеры.

```
apiVersion: v1
kind: Pod
metadata:
   name: hw-4
   namespace: default
spec:
   containers:
     - name: multitool
       image: wbitt/network-multitool
       ports:
        - containerPort: 8080
```

```
user@k8s:/opt/hw_k8s_4$ microk8s kubectl apply -f task-1-pod.yaml
pod/hw-4 created
user@k8s:/opt/hw_k8s_4$ microk8s kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
hw-4                   1/1     Running   0          5s
hw-4-7bc567b75-4qchr   2/2     Running   0          6m6s
hw-4-7bc567b75-fgqrb   2/2     Running   0          6m6s
hw-4-7bc567b75-s9vfw   2/2     Running   0          6m6s

```

```
user@k8s:/opt/hw_k8s_4$ microk8s kubectl get pods -o wide
NAME                   READY   STATUS    RESTARTS   AGE     IP           NODE   NOMINATED NODE   READINESS GATES
hw-4                   1/1     Running   0          7m18s   10.1.77.2    k8s    <none>           <none>
hw-4-7bc567b75-4qchr   2/2     Running   0          13m     10.1.77.60   k8s    <none>           <none>
hw-4-7bc567b75-fgqrb   2/2     Running   0          13m     10.1.77.62   k8s    <none>           <none>
hw-4-7bc567b75-s9vfw   2/2     Running   0          13m     10.1.77.61   k8s    <none>           <none>

user@k8s:/opt/hw_k8s_4$ microk8s kubectl exec -n default hw-4 -- curl 10.1.77.60:80
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   615  100   615    0     0   964k      0 --:--:-- --:--:-- --:--:--  600k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
user@k8s:/opt/hw_k8s_4$ microk8s kubectl exec -n default hw-4 -- curl 10.1.77.61:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   615  100   615    0     0   903k      0 --:--:-- --:--:-- --:--:--  600k
user@k8s:/opt/hw_k8s_4$ microk8s kubectl exec -n default hw-4 -- curl 10.1.77.62:80
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   615  100   615    0     0   195k      0 --:--:-- --:--:-- --:--:--  300k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
user@k8s:/opt/hw_k8s_4$

```

4. Продемонстрировать доступ с помощью `curl` по доменному имени сервиса.

```
user@k8s:/opt/hw_k8s_4$ microk8s kubectl exec -n default hw-4 -- curl hw-4-svc:9001
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   615  100   615    0     0   553k      0 --:--:-- --:--:-- --:--:--  600k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

5. Предоставить манифесты Deployment и Service в решении, а также скриншоты или вывод команды п.4.

[task-1-deployment.yaml](https://github.com/stepynin-georgy/hw_k8s_4/blob/main/task-1-deployment.yaml)

[task-1-pod.yaml](https://github.com/stepynin-georgy/hw_k8s_4/blob/main/task-1-pod.yaml)

[task-1-svc.yaml](https://github.com/stepynin-georgy/hw_k8s_4/blob/main/task-1-svc.yaml)

------

### Задание 2. Создать Service и обеспечить доступ к приложениям снаружи кластера

1. Создать отдельный Service приложения из Задания 1 с возможностью доступа снаружи кластера к nginx, используя тип NodePort.

```
apiVersion: v1
kind: Service
metadata:
  name: hw-4-2-svc
spec:
  selector:
    app: hw-4
  ports:
    - name: for-nginx
      nodePort: 30500
      targetPort: 80
      port: 80
    - name: for-multitool
      nodePort: 30501
      port: 8080
      targetPort: 8080
  type: NodePort
```

```
user@k8s:/opt/hw_k8s_4$ microk8s kubectl apply -f task-2-svc.yaml
service/hw-4-2-svc created
user@k8s:/opt/hw_k8s_4$ microk8s kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                       AGE
hw-4-2-svc   NodePort    10.152.183.126   <none>        80:30500/TCP,8080:30501/TCP   6s
hw-4-svc     ClusterIP   10.152.183.34    <none>        9001/TCP,9002/TCP             17m
kubernetes   ClusterIP   10.152.183.1     <none>        443/TCP                       7d
```

2. Продемонстрировать доступ с помощью браузера или `curl` с локального компьютера.

```
user@k8s:/opt/hw_k8s_4$ curl http://158.160.72.54:30500
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
user@k8s:/opt/hw_k8s_4$ curl http://158.160.72.54:30501
WBITT Network MultiTool (with NGINX) - hw-4-745df74468-jtrn8 - 10.1.77.8 - HTTP: 8080 , HTTPS: 443 . (Formerly praqma/network-multitool)
```

3. Предоставить манифест и Service в решении, а также скриншоты или вывод команды п.2.

![image](https://github.com/stepynin-georgy/hw_k8s_4/blob/main/img/Screenshot_142.png)

![image](https://github.com/stepynin-georgy/hw_k8s_4/blob/main/img/Screenshot_143.png)

[task-2-svc.yaml](https://github.com/stepynin-georgy/hw_k8s_4/blob/main/task-2-svc.yaml)

------

### Правила приёма работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.
