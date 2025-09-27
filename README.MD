## SRE - Mindbox task

### Мультизональный кластер (три зоны), в котором пять нод
k8s cluster 5 нод, 3 зоны, предположим, что распределение такое:
```azure
ZONE A - node1, node2
ZONE B - node3, node4
ZONE C - node5
```
В таком случае поды нужно расположить на каждую зону (возьмем 3 реплики, 4 - оставим на пиковую нагрузку по условию), чтобы обеспечить отказоустойчивость по зонам.

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      whenUnsatisfiable: ScheduleAnyway
      topologyKey: topology.kubernetes.io/zone
      labelSelector:
        matchLabels:
          app: mindbox
```
```azure
ZONE A - node1, node2 [somewhere pod1]
ZONE B - node3, node4 [somewhere pod2]
ZONE C - node5 [pod3]

Pod4 может быть запущен в одной из 3-ёх зон (учитывая свободные ресурсы и maxSkew=1)>
```

### Приложение требует около 5-10 секунд для инициализации
Для долгих стартов нужно обязательно использовать startupProbe. (так же ещё добавим readinessProbe и livenessProbe)


По условию мы имеем 5-10 секунд на старт, возьмем верхнюю границу для пробы - 10 секунд.
```azure
2 * 5 = 10 секунд | 2 попытки по 5 секунд | в лучшем случае мы поднимемся за 5 секунд
```

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: http
  failureThreshold: 2
  periodSeconds: 5
```

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: http
  initialDelaySeconds: 0
  periodSeconds: 5
  failureThreshold: 1
```

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: http
  periodSeconds: 10
  failureThreshold: 3
```

### На первые запросы приложению требуется значительно больше ресурсов CPU, в дальнейшем потребление ровное в районе 0.1 CPU. По памяти всегда “ровно” в районе 128M memory
Вначале скачок, но потом приложение потребляет в районе 0.1 CPU, установим request.cpu=100m, а limits.memory=300m на скачок ("значительно больше")

Потребление по памяти "всегда ровное" ~ 128M, установим request.memory на 128M, а limits.memory на 160M чуток с запасом. ("ровное")
```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "160Mi"
    cpu: "300m"
```

### Приложение имеет дневной цикл по нагрузке – ночью запросов на порядки меньше, пик – днём
В таком случае нужно скейлиться днём, а ночью снижать потребление ресурсов.

Возьмем диапазон времени: от 7:00 до 23:00 - день, от 23:00 до 7:00 - ночь.

И сделаем просто через cronjob и job-ресурсы, будем запускать джобу в 7:00 и 23:00 для скейлинга.

По условию нам хватает 4 подов на пиковую нагрузку, значит нужно держать 4 пода днём, а ночью 3.

```yaml
#mindbox-scale-up-pods
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mindbox-scale-up-pods
spec:
  schedule: "0 7 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: mindbox-service-account
          containers:
            - name: kubectl
              image: bitnami/kubectl
              command:
                - /bin/sh
                - -c
                - kubectl scale deployment mindbox-deployment --replicas=4
          restartPolicy: OnFailure
```

```yaml
#mindbox-scale-down-pods
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mindbox-scale-down-pods
spec:
  schedule: "0 23 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: mindbox-service-account
          containers:
            - name: kubectl
              image: bitnami/kubectl
              command:
                - /bin/sh
                - -c
                - kubectl scale deployment mindbox-deployment --replicas=3
          restartPolicy: OnFailure
```

Но, как понять, сейчас день или ночь при деплое? Можно использовать обычную джобу со скриптом для инициализации.
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mindbox-initial-scale
spec:
  template:
    spec:
      serviceAccountName: mindbox-service-account
      containers:
        - name: kubectl
          image: bitnami/kubectl
          command:
            - /bin/sh
            - -c
            - |
              HOUR=$(date +%H)
              if [ $HOUR -ge 7 ] && [ $HOUR -lt 23 ]; then
                kubectl scale deployment mindbox-deployment --replicas=4
              else
                kubectl scale deployment mindbox-deployment --replicas=3
              fi
      restartPolicy: OnFailure
```

Также для задания "Приложение имеет дневной цикл по нагрузке..." были созданы следующие ресурсы: 
* ServiceAccount
* ClusterRole
* ClusterRolebinding
* CronJob
* Job

Все манифесты можно посмотреть тут.