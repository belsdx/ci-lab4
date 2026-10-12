# Лабораторная работа №4. Оркестрация в многоконтейнерных системах

## 1. Цель работы

Изучение создания кластера и работы с Kubernetes: развёртывание локального
кластера при помощи `kind`, знакомство с основными ресурсами (namespace, Pod,
Volume, ReplicaSet) и развёртывание приложения с обновлением без простоя
(Deployment, Service).

## 2. Вариант задания

Выполнены все три задания (для групп 10–12):
1. Подготовка окружения для локального кластера.
2. Знакомство с ресурсами Kubernetes.
3. Работа с Kubernetes (развёртывание приложения).

## 3. Код приложений и конфигурационных файлов

<!--
В этом разделе разместите ключевые фрагменты (полные файлы — в репозитории):
- task1/kind-config.yaml
- task2/pod-with-volume.yaml
- task2/replicaset.yaml
- task3/app/app.py, task3/app/Dockerfile
- task3/deployment.yaml, task3/service.yaml
и скриншоты по каждому шагу (см. docs/任务书.md, места помечены 📸).
-->

### 3.1. Конфигурация кластера (`task1/kind-config.yaml`)

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: lab4-cluster
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

### 3.2. Pod с томом (`task2/pod-with-volume.yaml`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-demo
spec:
  containers:
    - name: writer
      image: busybox
      command: ["/bin/sh", "-c"]
      args: ["echo 'Hello from volume' > /data/hello.txt; sleep 3600"]
      volumeMounts:
        - name: demo-volume
          mountPath: /data
    - name: reader
      image: busybox
      command: ["/bin/sh", "-c"]
      args: ["sleep 5; cat /data/hello.txt; sleep 3600"]
      volumeMounts:
        - name: demo-volume
          mountPath: /data
  volumes:
    - name: demo-volume
      emptyDir: {}
```

### 3.3. ReplicaSet (`task2/replicaset.yaml`)

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

### 3.4. Приложение (`task3/app/app.py`)

```python
from flask import Flask

app = Flask(__name__)


@app.route("/")
def hello():
    return "Hello from version 1.0\n"


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### 3.5. Deployment (`task3/deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5000
```

### 3.6. Service (`task3/service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 5000
      protocol: TCP
  type: NodePort
```

### 3.7. Результаты выполнения (скриншоты)

<!--
Вставьте сюда скриншоты 2–16 (см. docs/任务书.md):
- kubectl get nodes -o wide
- kubectl get pods -A
- kubectl api-resources / get namespaces
- логи volume-demo -c reader
- ReplicaSet + самовосстановление
- Deployment + Service + доступ
- rolling update v1->v2 + откат
-->

## 4. Ответы на контрольные вопросы

### 1. Что такое Kubernetes и зачем он нужен в работе с контейнерами?

Kubernetes (K8s) — это open-source платформа для автоматизации развёртывания,
масштабирования и управления контейнеризированными приложениями. В работе с
контейнерами она нужна, чтобы управлять не отдельными контейнерами, а их
множеством: планировать запуск по нодам, следить за состоянием, автоматически
перезапускать упавшие, масштабировать под нагрузку и предоставлять сетевую
связность и балансировку.

### 2. Отличие «запустить контейнеры вручную через Docker» от «управлять ими в Kubernetes».

Docker запускает контейнеры на **одной** машине; вы сами выбираете порты,
объёмы, порядок запуска и вручную перезапускаете упавшие. Kubernetes управляет
контейнерами **на кластере из многих машин**: он сам решает, на какой ноде
запустить под, следит за желаемым состоянием (declarative), автоматически
перезапускает, масштабирует и балансирует нагрузку.

### 3. 1–2 примера задач, где оркестрация экономит время.

- Горизонтальное масштабирование: увеличить число реплик веб-приложения одной
  командой `kubectl scale` вместо ручного запуска контейнеров на нескольких
  машинах.
- Самовосстановление: если под упал или нода вышла из строя, Kubernetes
  автоматически перезапустит/переместит под без вмешательства человека.

### 4. Основные компоненты кластера Kubernetes и их назначение.

- **kube-apiserver** — главный API-шлюз, через который идут все команды.
- **etcd** — распределённое хранилище состояния кластера (ключ-значение).
- **kube-controller-manager** — запускает контроллеры (ReplicaSet, Deployment и др.).
- **kube-scheduler** — выбирает ноду для запуска пода.
- **kubelet** — агент на каждой ноде, запускающий контейнеры.
- **kube-proxy** — сетевые правила для сервисов.
- **Container runtime** (containerd, CRI-O) — непосредственно запускает контейнеры.

### 5. Как создать под с помощью YAML-манифеста? Обязательные поля.

Командой `kubectl apply -f pod.yaml`. Обязательные поля:
- **apiVersion** — версия API (например, `v1` для Pod).
- **kind** — тип ресурса (`Pod`).
- **metadata** — имя и метки (`name`, `labels`).
- **spec** — описание желаемого состояния (в частности `containers`).

### 6. YAML-файл (10–15 строк) простого пода + комментарий 3–4 полей.

```yaml
apiVersion: v1            # версия API для Pod
kind: Pod                 # тип ресурса
metadata:
  name: my-pod            # имя пода
spec:
  containers:
    - name: nginx         # имя контейнера
      image: nginx:1.25   # образ
      ports:
        - containerPort: 80
```

Комментарии: `apiVersion` — версия API; `kind` — тип; `metadata.name` — имя;
`spec.containers` — список контейнеров с образом и портами. Если пропустить
обязательное поле (например, `spec`), `kubectl apply` вернёт ошибку валидации и
под не будет создан.

### 7. Сервисы в Kubernetes; разница ClusterIP и Headless.

Service — это стабильная сетевая точка доступа к набору подов (постоянный IP и
DNS-имя), поскольку поды недолговечны и получают новые IP при пересоздании.
- **ClusterIP** — виртуальный IP внутри кластера, балансирует трафик между
  подами по selector.
- **Headless** (`clusterIP: None`) — не выделяет единый IP; DNS возвращает IP
  **каждого** пода напрямую. Нужен, когда клиент сам должен выбрать под
  (например, stateful-приложения, базы данных).

### 8. Отличие NodePort и LoadBalancer.

- **NodePort** — открывает порт (30000–32767) на **каждой** ноде кластера и
  пробрасывает трафик в Service. Доступ: `<node-ip>:<nodePort>`.
- **LoadBalancer** — создаёт внешний балансировщик (у облачного провайдера),
  который выделяет внешний IP и направляет трафик в Service. Доступ:
  `<external-ip>:<port>`.

Пошагово (NodePort): пользователь → `<node-ip>:<nodePort>` → kube-proxy →
Service (ClusterIP) → под. (LoadBalancer): пользователь → внешний IP балансировщика
→ балансировщик → Service → под.

### 9. Порты NodePort и ограничения.

NodePort использует диапазон **30000–32767** (настраивается через
`--service-node-port-range`). Ограничения: порт должен быть свободен на всех
нодах, нельзя использовать стандартные порты (80/443) напрямую, один NodePort —
один Service.

### 10. Когда достаточно NodePort, а когда нужен LoadBalancer.

- **NodePort достаточно** — для разработки, тестирования, доступа с локальной
  машины или внутри небольшой инфраструктуры.
- **LoadBalancer нужен** — в продакшене, когда нужен стабильный внешний IP,
  автоматическое распределение по нодам и интеграция с облачным балансировщиком.

### 11. Ingress и какую проблему он решает.

Ingress — ресурс, управляющий HTTP/HTTPS-маршрутизацией внешнего трафика к
сервисам по правилам (хост, путь). Решает проблему «много сервисов — много
портов»: вместо отдельного NodePort/LoadBalancer на каждый сервис, один
Ingress маршрутизирует по URL к нужным сервисам.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port: { number: 80 }
```

Трафик с `app.example.com/api` направляется в сервис `api-service`.

### 12. Сравнение Deployment, StatefulSet, DaemonSet.

| Критерий | Deployment | StatefulSet | DaemonSet |
|----------|-----------|-------------|-----------|
| Цель | обычные stateless-приложения | stateful-приложения с постоянной идентичностью | один под на каждую ноду |
| Создание подов | неупорядоченно, имена случайные | упорядоченно, стабильные имена (`pod-0`, `pod-1`) | по одному на ноду |
| Привязка к узлам | нет | может (stable storage) | да, к каждой ноде |

Примеры: Deployment — веб-приложение; StatefulSet — база данных (PostgreSQL);
DaemonSet — агент логирования (Promtail/Fluentd). При масштабировании до 3
реплик: Deployment создаст 3 произвольных пода; StatefulSet — упорядоченно
`-0,-1,-2` с сохранением томов; DaemonSet — по одному поду на каждой из 3 нод
(не масштабируется числом реплик).

### 13. Разница Job и CronJob.

- **Job** — запускает задачу **один раз** до успешного завершения (batch).
- **CronJob** — запускает Job по расписанию (cron-выражение).

Примеры: Job — однократная миграция БД; CronJob — ежедневное резервное
копирование. Проверка: `kubectl get jobs`, `kubectl describe job <name>`,
`kubectl logs job/<name>` (статус и логи).

### 14. ConfigMap и Secret; 2 способа передачи данных в под.

- **ConfigMap** — несекретные конфигурационные данные (ключ-значение).
- **Secret** — секретные данные (пароли, токены), хранятся в base64.

Способы передачи: (1) как переменные окружения (`envFrom`/`valueFrom`);
(2) как смонтированный том (файлы). Переменные окружения удобны для
небольших настроек; том — когда много значений или нужно обновлять без
перезапуска пода.

### 15. YAML-фрагменты ConfigMap и Secret; почему пароли нельзя в ConfigMap.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
```
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: cGFzc3dvcmQ=   # base64
```

ConfigMap хранит данные в открытом виде и не предназначен для секретов;
Secret хотя бы кодируется (base64) и может шифроваться в etcd, а также
контролируется через RBAC. Пароли в ConfigMap — риск утечки.

### 16. 8–10 основных команд kubectl по группам.

- **Посмотреть состояние:** `kubectl get pods/nodes/services`, `kubectl describe <res>`,
  `kubectl logs <pod>`.
- **Создать/обновить:** `kubectl apply -f file.yaml`, `kubectl create`, `kubectl set image`.
- **Удалить:** `kubectl delete <res> <name>`, `kubectl delete -f file.yaml`.
- **Отладить:** `kubectl describe <res>`, `kubectl logs -f <pod>`, `kubectl exec -it <pod> -- sh`.

### 17. Namespaces и их назначение; примеры.

Namespace — логическое разделение ресурсов внутри кластера на изолированные
группы. Используется для изоляции, организации и контроля доступа. Примеры:
`dev`/`stage`/`prod` (разные окружения), разные команды/проекты, изоляция
по клиентам.

### 18. Как создать namespace и выполнять команды только в нём.

```bash
kubectl create namespace my-ns
# или манифестом
kubectl apply -f ns.yaml
# команда в конкретном namespace
kubectl get pods -n my-ns
# переключить контекст по умолчанию
kubectl config set-context --current --namespace=my-ns
```

### 19. Пробы (probes): livenessProbe, readinessProbe, startupProbe.

- **livenessProbe** — жив ли контейнер; если нет — перезапустить.
- **readinessProbe** — готов ли контейнер принимать трафик; если нет — убрать
  из Service.
- **startupProbe** — защита медленно стартующих приложений (пока не пройдёт,
  liveness/readiness не запускаются).

### 20. PV и PVC; как выделяется место под хранилище.

- **PV (PersistentVolume)** — физический том (диск) в кластере, выделенный
  администратором или динамически.
- **PVC (PersistentVolumeClaim)** — запрос пользователя на хранилище
  (размер, режим доступа).

Место выделяется так: пользователь создаёт PVC → Kubernetes подбирает
подходящий PV (или Provisioner динамически создаёт его) → PVC связывается с PV,
и под монтирует том через PVC.

### 21. Реакция Kubernetes на падение пода/узла; механизмы самовосстановления.

- **Падение пода:** kubelet обнаруживает сбой, ReplicaSet/Deployment
  перезапускает под (livenessProbe, restartPolicy).
- **Падение узла:** kube-scheduler перепланирует поды на другую ноду
  (controller-manager помечает ноду NotReady и пересоздаёт поды).
- Механизмы: liveness/readiness-пробы, репликация (ReplicaSet), самовосстановление
  через контроллеры, планировщик на замену.

### 22. Обновление без простоя; rolling update и параметры.

Rolling update — стратегия, при которой новые реплики запускаются постепенно,
а старые удаляются по мере готовности, чтобы приложение оставалось доступным.
Параметры Deployment: `strategy.rollingUpdate.maxSurge` (сколько доп. подов можно
создать сверх желаемого числа) и `maxUnavailable` (сколько подов может быть
недоступно во время обновления).

### 23. Как запустить обновление и следить за прогрессом.

```bash
kubectl set image deployment/myapp-deployment myapp=myapp:v2
kubectl rollout status deployment/myapp-deployment
kubectl rollout history deployment/myapp-deployment
# откат
kubectl rollout undo deployment/myapp-deployment
```
Прогресс отслеживается через `kubectl rollout status` (ждёт завершения) и
`kubectl get pods -w` (наблюдение за сменой подов).
