
# Выполнено ДЗ №2

# Kubernetes controllers. ReplicaSet, Deployment, DaemonSet

 - [x] Основное ДЗ
 - [x] Задание со *

## В процессе сделано:

__Цели домашнего задания__
1) Научиться создавать и конфигурировать Replicaset, Deployment для своего приложения;
2) Научиться управлять обновлением своего приложения;
3) Научиться использовать механизм Probes для проверки
работоспособности своих приложений.

__1 Выполнение:__
_Работу и демонстрацию продолжаю в установленном кластере kubernetes в ОС Talos._

Удалю и заново создам namespace homework в kubernetes. Создаю манифесты.
#### ReplicaSet

___Запускает 3 экземпляра пода, полностью аналогичных по спецификации прошлому ДЗ.___ 

Создаю и применяю манифест hw-replicaset.yaml
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: hw
  namespace: homework
  labels:
    app: hw
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hw
  template:
    metadata:
      labels:
        app: hw
    spec:
      containers:
      - name: server
        image: zagretdinov/hw:0.3
        env:
          - name: PRODUCT_CATALOG_SERVICE_ADDR
            value: "productcatalogservice:3550"
          - name: CURRENCY_SERVICE_ADDR
            value: "currencyservice:7000"
          - name: CART_SERVICE_ADDR
            value: "cartservice:7070"
          - name: RECOMMENDATION_SERVICE_ADDR
            value: "recommendationservice:8080"
          - name: SHIPPING_SERVICE_ADDR
            value: "shippingservice:50051"
          - name: CHECKOUT_SERVICE_ADDR
            value: "checkoutservice:5050"
          - name: AD_SERVICE_ADDR
            value: "adservice:9555"
```
В результате вывод команды kubectl get pods -l app=hw -n homework должен показывать, что запущена одна реплика микросервиса hw:
```
NAME       READY   STATUS    RESTARTS   AGE
hw-qq5kf   1/1     Running   0          3m32s
```
Одна работающая реплика - это уже неплохо, но в задании, требуется создание 3 инстанса одного и того же сервиса для:
- Повышения отказоустойчивости
- Распределения нагрузки между репликами

```
devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectl scale replicaset hw --replicas=3 -n homework
replicaset.apps/hw scaled
devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectl get pods -l app=hw -n homework
NAME       READY   STATUS    RESTARTS   AGE
hw-2tpg6   1/1     Running   0          29s
hw-7wxcm   1/1     Running   0          29s
hw-qq5kf   1/1     Running   0          11m
```
можно проверить следующим образом:
```
devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectl get rs hw -n homework
NAME   DESIRED   CURRENT   READY   AGE
hw     3         3         3       17m
```
Благодаря контроллеру pod'ы восстанавливаются после их ручного удаления
```
^Cdevops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectl delete pods -l app=hw -n homework | kubectl get pods -l app=hw -n homework -w 
NAME       READY   STATUS        RESTARTS   AGE
hw-8rstq   1/1     Terminating   0          61s
hw-m9l9x   1/1     Terminating   0          61s
hw-n6vzf   1/1     Running       0          61s
hw-vwpmf   0/1     Pending       0          0s
hw-n6vzf   1/1     Terminating   0          61s
hw-vwpmf   0/1     Pending       0          0s
hw-vwpmf   0/1     Pending       0          0s
hw-hmzwx   0/1     Pending       0          0s
hw-vwpmf   0/1     ContainerCreating   0          0s
hw-hmzwx   0/1     Pending             0          0s
hw-hmzwx   0/1     Pending             0          0s
hw-hmzwx   0/1     ContainerCreating   0          0s
hw-vwpmf   0/1     ContainerCreating   0          0s
hw-hmzwx   0/1     ContainerCreating   0          0s
hw-lwb9j   0/1     Pending             0          0s
hw-lwb9j   0/1     Pending             0          0s
hw-lwb9j   0/1     Pending             0          0s
hw-lwb9j   0/1     ContainerCreating   0          0s
hw-lwb9j   0/1     ContainerCreating   0          0s
hw-8rstq   0/1     Terminating         0          61s
hw-m9l9x   0/1     Terminating         0          61s
hw-n6vzf   0/1     Terminating         0          61s
hw-8rstq   0/1     Terminating         0          61s
hw-8rstq   0/1     Terminating         0          61s
hw-n6vzf   0/1     Terminating         0          61s
hw-8rstq   0/1     Terminating         0          61s
hw-n6vzf   0/1     Terminating         0          61s
hw-n6vzf   0/1     Terminating         0          61s
hw-m9l9x   0/1     Terminating         0          62s
hw-lwb9j   1/1     Running             0          1s
hw-m9l9x   0/1     Terminating         0          62s
hw-m9l9x   0/1     Terminating         0          62s
hw-vwpmf   1/1     Running             0          1s
hw-hmzwx   1/1     Running             0          1s
```

- Повторно применим манифест frontend-replicaset.yaml
- Убедимся, что количество реплик вновь уменьшилось до одной

```
devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectapply -f hw-replicaset.yaml
replicaset.apps/hw configured
devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectl get pods -l app=hw -n homework
NAME       READY   STATUS    RESTARTS   AGE
hw-lwb9j   1/1     Running   0          25m
devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectl apply -f hw-replicaset.yaml
replicaset.apps/hw configured
devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectl get pods -l app=hw -n homework
NAME       READY   STATUS    RESTARTS   AGE
hw-lwb9j   1/1     Running   0          37m
hw-pzwpf   1/1     Running   0          3s
hw-wkx5g   1/1     Running   0          3s
```
___В дополнение к этому будет иметь readiness пробу, проверяющую наличие файла /homework/index.html___

Здесь добавлена секция readinessProbe, которая будет выполнена внутри контейнера и проверит наличие файла /homework/index.html. Опции initialDelaySeconds, periodSeconds, timeoutSeconds, successThreshold и failureThreshold настраивают параметры проверки готовности.

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: hw
  namespace: homework
  labels:
    app: hw
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hw
  template:
    metadata:
      labels:
        app: hw
    spec:
      containers:
      - name: server
        image: zagretdinov/hw:0.4
        ports:
        - containerPort: 8000
        readinessProbe:
          exec:
            command:
            - cat
            - /homework/index.html
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        env:
          - name: PRODUCT_CATALOG_SERVICE_ADDR
            value: "productcatalogservice:3550"
          - name: CURRENCY_SERVICE_ADDR
            value: "currencyservice:7000"
          - name: CART_SERVICE_ADDR
            value: "cartservice:7070"
          - name: RECOMMENDATION_SERVICE_ADDR
            value: "recommendationservice:8080"
          - name: SHIPPING_SERVICE_ADDR
            value: "shippingservice:50051"
          - name: CHECKOUT_SERVICE_ADDR
            value: "checkoutservice:5050"
          - name: AD_SERVICE_ADDR
            value: "adservice:9555"
```
Для проверки того, что секция readinessProbe

```
devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectl describe rs hw -n homework
Name:         hw
Namespace:    homework
Selector:     app=hw
Labels:       app=hw
Annotations:  <none>
Replicas:     3 current / 3 desired
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=hw
  Containers:
   server:
    Image:      zagretdinov/hw:0.4
    Port:       8000/TCP
    Host Port:  0/TCP
    Readiness:  exec [cat homework/index.html] delay=5s timeout=5s period=10s #success=1 #failure=3
    Environment:
      PRODUCT_CATALOG_SERVICE_ADDR:  productcatalogservice:3550
      CURRENCY_SERVICE_ADDR:         currencyservice:7000
      CART_SERVICE_ADDR:             cartservice:7070
      RECOMMENDATION_SERVICE_ADDR:   recommendationservice:8080
      SHIPPING_SERVICE_ADDR:         shippingservice:50051
      CHECKOUT_SERVICE_ADDR:         checkoutservice:5050
      AD_SERVICE_ADDR:               adservice:9555
    Mounts:                          <none>
  Volumes:                           <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  7m1s  replicaset-controller  Created pod: hw-hj8hf
  Normal  SuccessfulCreate  7m1s  replicaset-controller  Created pod: hw-2gmzr
  Normal  SuccessfulCreate  7m1s  replicaset-controller  Created pod: hw-fbk4f
```
Проверяю работу готовности внутри контейнера.

```
devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectl exec -it hw-2gmzr -n homework -- cat homework/index.html
<!DOCTYPE html>
<html>
<head>
<title>Welcome devops!</title>
<html>
<body>
<h1>Welcome devops!</h1>
<p>Homework page 1</p>
</body>
</html>devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ 
```

Изменяю поле kind с ReplicaSet на Deployment и создаю hw-deployment.yaml.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hw-service
  namespace: homework
  labels:
    app: hw-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hw-service
  template:
    metadata:
      labels:
        app: hw-service
    spec:
      containers:
      - name: hw-service
        image: zagretdinov/hw:0.4
        ports:
        - containerPort: 8000
        env:
        - name: PORT
          value: "50051"
        - name: DISABLE_PROFILER
          value: "1"
```

```
devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectl get deploy -n homework
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
hw-service   3/3     3            3           3m54s
devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectl get rs -n homework
NAME                    DESIRED   CURRENT   READY   AGE
hw                      3         3         3       36m
hw-service-6975cb869c   3         3         3       4m
```




Добавляю секцию strategy, где устанавливается тип обновления (type: RollingUpdate) и определяется параметр maxUnavailable. В данном случае он установлен в 1, что означает, что в процессе обновления может быть недоступен только один под.

После применения этого файла конфигурации, Kubernetes будет обновлять манифест таким образом, чтобы в любой момент времени не более одного пода было недоступно.

```
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: hw
```
```

devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectl describe rs hw-service -n homework
Name:           hw-service-6975cb869c
Namespace:      homework
Selector:       app=hw-service,pod-template-hash=6975cb869c
Labels:         app=hw-service
                pod-template-hash=6975cb869c
Annotations:    deployment.kubernetes.io/desired-replicas: 3
                deployment.kubernetes.io/max-replicas: 4
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/hw-service
Replicas:       2 current / 2 desired
Pods Status:    2 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=hw-service
           pod-template-hash=6975cb869c
  Containers:
   hw-service:
    Image:      zagretdinov/hw:0.4
    Port:       8000/TCP
    Host Port:  0/TCP
    Environment:
      PORT:              50051
      DISABLE_PROFILER:  1
    Mounts:              <none>
  Volumes:               <none>
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  42m    replicaset-controller  Created pod: hw-service-6975cb869c-jmkwn
  Normal  SuccessfulCreate  42m    replicaset-controller  Created pod: hw-service-6975cb869c-nm2nw
  Normal  SuccessfulCreate  42m    replicaset-controller  Created pod: hw-service-6975cb869c-cxtv7
  Normal  SuccessfulDelete  4m21s  replicaset-controller  Deleted pod: hw-service-6975cb869c-cxtv7

Name:           hw-service-766b989444
Namespace:      homework
Selector:       app=hw-service,pod-template-hash=766b989444
Labels:         app=hw-service
                pod-template-hash=766b989444
Annotations:    deployment.kubernetes.io/desired-replicas: 3
                deployment.kubernetes.io/max-replicas: 4
                deployment.kubernetes.io/revision: 2
Controlled By:  Deployment/hw-service
Replicas:       2 current / 2 desired
Pods Status:    0 Running / 2 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=hw-service
           pod-template-hash=766b989444
  Containers:
   hw-service:
    Image:      zagretdinov/hw:0.4
    Port:       8000/TCP
    Host Port:  0/TCP
    Environment:
      PORT:              50051
      DISABLE_PROFILER:  1
    Mounts:              <none>
  Volumes:               <none>
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  4m21s  replicaset-controller  Created pod: hw-service-766b989444-gpg9l
  Normal  SuccessfulCreate  4m21s  replicaset-controller  Created pod: hw-service-766b989444-cgztl
  ```
### Обновление Deployment

попробую обновить наш Deployment на версию образа 0.4
```
   containers:
      - name: hw-service
        image: zagretdinov/hw:0.4
```

- Создание одного нового pod с версией образа 0.4
- Удаление одного из старых pod
- Создание еще одного нового pod
```
kubectl apply -f hw-deployment.yaml | kubectl get pods -l app=hw-service -w -n homework
NAME                          READY   STATUS    RESTARTS   AGE
hw-service-85948648b6-4dn92   1/1     Running   0          28s
hw-service-85948648b6-9ptnk   1/1     Running   0          28s
hw-service-85948648b6-dnmc9   1/1     Running   0          28s
hw-service-6975cb869c-zx9p6   0/1     Pending   0          0s
hw-service-6975cb869c-zx9p6   0/1     Pending   0          0s
hw-service-6975cb869c-zx9p6   0/1     Pending   0          0s
hw-service-6975cb869c-zx9p6   0/1     ContainerCreating   0          0s
hw-service-6975cb869c-zx9p6   0/1     ContainerCreating   0          0s
hw-service-6975cb869c-zx9p6   1/1     Running             0          2s
hw-service-85948648b6-9ptnk   1/1     Terminating         0          30s
hw-service-6975cb869c-8gbgf   0/1     Pending             0          0s
hw-service-6975cb869c-8gbgf   0/1     Pending             0          0s
hw-service-6975cb869c-8gbgf   0/1     Pending             0          0s
hw-service-6975cb869c-8gbgf   0/1     ContainerCreating   0          0s
hw-service-6975cb869c-8gbgf   0/1     ContainerCreating   0          0s
hw-service-85948648b6-9ptnk   0/1     Terminating         0          31s
hw-service-85948648b6-9ptnk   0/1     Terminating         0          31s
hw-service-85948648b6-9ptnk   0/1     Terminating         0          31s
hw-service-85948648b6-9ptnk   0/1     Terminating         0          31s
hw-service-6975cb869c-8gbgf   1/1     Running             0          2s
hw-service-85948648b6-dnmc9   1/1     Terminating         0          32s
hw-service-6975cb869c-5d29t   0/1     Pending             0          0s
hw-service-6975cb869c-5d29t   0/1     Pending             0          0s
hw-service-6975cb869c-5d29t   0/1     Pending             0          0s
hw-service-6975cb869c-5d29t   0/1     ContainerCreating   0          0s
hw-service-6975cb869c-5d29t   0/1     ContainerCreating   0          0s
hw-service-85948648b6-dnmc9   0/1     Terminating         0          33s
hw-service-85948648b6-dnmc9   0/1     Terminating         0          33s
hw-service-85948648b6-dnmc9   0/1     Terminating         0          33s
hw-service-85948648b6-dnmc9   0/1     Terminating         0          33s
hw-service-6975cb869c-5d29t   1/1     Running             0          1s
hw-service-85948648b6-4dn92   1/1     Terminating         0          33s
hw-service-85948648b6-4dn92   0/1     Terminating         0          34s
hw-service-85948648b6-4dn92   0/1     Terminating         0          34s
hw-service-85948648b6-4dn92   0/1     Terminating         0          34s
hw-service-85948648b6-4dn92   0/1     Terminating         0          34s
```
Убедимся что:

    Все новые pod развернуты из образа 0.4
    Создано два ReplicaSet:
        Один (новый) управляет тремя репликами pod с образом 0.4
        Второй (старый) управляет нулем реплик pod с образом 0.3


```
devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectl get rs -n homework
NAME                    DESIRED   CURRENT   READY   AGE
hw                      3         3         3       135m
hw-service-6975cb869c   3         3         3       99s
hw-service-85948648b6   0         0         0       2m8s
devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectl rollout history deployment hw-service -n homework
deployment.apps/hw-service 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```
### Задание с *

___Добавить к манифесту deployment-а спецификацию, обеспечивающую запуск подов деплоймента, только на нодах кластера, имеющих метку homework=true___

Чтобы обеспечить запуск подов Deployment только на узлах кластера, имеющих метку homework=true, использую селектор узлов (nodeSelector) в спецификации манифеста
```
spec:
      nodeSelector:
        homework: "true"
      containers:
      - name: hw-service
        image: zagretdinov/hw:0.4
```
Здесь добавлена секция nodeSelector, которая указывает Kubernetes, что поды этого Deployment должны быть запущены только на узлах, у которых есть метка homework=true.
После применения этого файла конфигурации, Kubernetes будет автоматически размещать поды вашего Deployment только на узлах с меткой homework=true.

В результате до применения манифеста:
```
devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectl get pods -o wide -n homework
NAME                          READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
hw-2gmzr                      1/1     Running   0          83m   10.244.0.85   talos2   <none>           <none>
hw-fbk4f                      1/1     Running   0          83m   10.244.0.86   talos3   <none>           <none>
hw-hj8hf                      1/1     Running   0          83m   10.244.0.84   talos1   <none>           <none>
hw-service-6975cb869c-6vfzf   1/1     Running   0          58s   10.244.0.99   talos3   <none>           <none>
hw-service-6975cb869c-jmkwn   1/1     Running   0          51m   10.244.0.90   talos1   <none>           <none>
hw-service-6975cb869c-nm2nw   1/1     Running   0          51m   10.244.0.91   talos2   <none>           <none>
```
и после применения.
```
devops@devops:~/zagretdinov-d_repo/kubernetes-controllers$ kubectl get pods -o wide -n homework
NAME                          READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
hw-2gmzr                      1/1     Running   0          84m   10.244.0.85   talos2   <none>           <none>
hw-fbk4f                      1/1     Running   0          84m   10.244.0.86   talos3   <none>           <none>
hw-hj8hf                      1/1     Running   0          84m   10.244.0.84   talos1   <none>           <none>
hw-service-6975cb869c-jmkwn   1/1     Running   0          51m   10.244.0.90   talos1   <none>           <none>
hw-service-6975cb869c-nm2nw   1/1     Running   0          51m   10.244.0.91   talos2   <none>           <none>
hw-service-766b989444-858l6   0/1     Pending   0          8s    <none>        <none>   <none>           <none>
hw-service-766b989444-m4rhm   0/1     Pending   0          8s    <none>        <none>   <none>           <none>
```
## Как запустить проект:
 - Проверяю под с помощью port-forward.
 ```
kubectl -n homework port-forward pod/hw 8001:8000
Forwarding from 127.0.0.1:8001 -> 8000
Forwarding from [::1]:8001 -> 8000
Handling connection for 8001
```
## Как проверить работоспособность:
 - Cсылка http://127.0.0.1:8001/homework/index.xtml

## PR checklist:
 - [x] Выставлен label с темой домашнего задания