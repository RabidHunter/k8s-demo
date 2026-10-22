# Звіт з Практичної роботи №10: Kubernetes (Рівень 60 балів)

---

## 1. Підготовка репозиторію та локальний запуск

Сервіс було успішно запущено локально за допомогою команди `npm run dev`.

### Опис відповідей ендпоінтів:
* **`/api/status`**: Повертає {"status":"ok","appMode":"dev","version":"1.0.0","timestamp":"2026-10-22T15:44:36.161Z"}
* **`/api/items?filter=a`**: Повертає [{"id":1,"name":"Item A"}]
* **`/api/slow`**: Повертає {"status":"ok","delayMs":2000}
* **`/api/error`**: Повертає {"status":"ok","message":"No error"}
* **`/healthz`**: Повертає ok
* **`/ready`**: Повертає {"status":"ready"}

---

## 2. Контейнеризація застосунку

### Команди:

* **Збірка образу:**
  ```bash
  docker build -t k8s-demo-app:1.0.0 .
  ```

* **Запуск контейнера:**
  ```bash
  docker run --rm -p 8080:8080 -e SECRET_TOKEN=local k8s-demo-app:1.0.0
  ```

### Приклад відповіді ендпоінту `/api/status`:
```bash
StatusCode        : 200
StatusDescription : OK
Content           : {"status":"ok","appMode":"dev","version":"1.0.0","timestamp":"2026-10-22T15:47:47.151Z"}
RawContent        : HTTP/1.1 200 OK
                    Connection: keep-alive
                    Keep-Alive: timeout=5
                    Content-Length: 88
                    Content-Type: application/json; charset=utf-8
                    Date: Thu, 22 Oct 2026 15:47:47 GMT
                    ETag: W/"58-3Zkl1BZhIstgoO7NsYa5...
Forms             : {}
Headers           : {[Connection, keep-alive], [Keep-Alive, timeout=5], [Content-Length, 88], [Content-Type, 
                    application/json; charset=utf-8]...}
Images            : {}
InputFields       : {}
Links             : {}
ParsedHtml        : mshtml.HTMLDocumentClass
RawContentLength  : 88
```

---

## 3. Запуск Minikube та перевірка кластера

* Активний контекст `kubectl`: `minikube` (перевірено за допомогою `kubectl config current-context`).
* Статус ноди Minikube: `Ready` (перевірено через `kubectl get nodes`).

### Вивід перевірки вузла:
```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   23m   v1.37.0
```

---

## 4. Розгортання ресурсів у Kubernetes

Маніфести було застосовано за допомогою `kubectl apply -f k8s/`. Всередині Namespace `k8s-demo` всі Pod-и знаходяться в статусі `Running` та `READY 1/1`.

### Вивід команди `kubectl get deployment,rs,pods,svc -n k8s-demo`:
```text
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8s-demo-api   2/2     2            2           25m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/k8s-demo-api-544d799d47   0         0         0       16m
replicaset.apps/k8s-demo-api-5c45559886   0         0         0       25m
replicaset.apps/k8s-demo-api-64b99988f7   2         2         2       16m
replicaset.apps/k8s-demo-api-7d69b4cbc5   0         0         0       22m

NAME                                READY   STATUS    RESTARTS   AGE
pod/k8s-demo-api-64b99988f7-n6dvr   1/1     Running   0          16m
pod/k8s-demo-api-64b99988f7-qm9kn   1/1     Running   0          16m

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/k8s-demo-api   ClusterIP   10.101.163.42   <none>        80/TCP    25m
```

---

## 5. Доступ до сервісу через Service та `kubectl port-forward`

Для отримання доступу виконано прокидання портів:
```bash
kubectl port-forward svc/k8s-demo-api -n k8s-demo 8080:80
```

### Шлях проходження запиту:
При використанні `kubectl port-forward` запит проходить за наступним ланцюжком:  
**Клієнт (curl / браузер)** - **Локальний порт (8080)** - **Процес kubectl** - **Kubernetes API Server** - **Kubelet на ноді** - **Pod (контейнер застосунку)**.

---

## 6. Відповіді на контрольні запитання

### a. У чому відмінність між ConfigMap та Secret?
* **ConfigMap** призначений для збереження неконфіденційних даних у відкритому текстовому вигляді (наприклад, конфігураційні файли, змінні середовища, URL-адреси).
* **Secret** використовується для збереження чутливої інформації (паролі, токени доступу, SSL-сертифікати). Дані в Secret кодуються у форматі Base64 і мають жорсткіші обмеження щодо прав доступу на рівні RBAC.

### b. У чому відмінність між liveness- та readiness-перевірками?
* **Liveness Probe:** перевіряє, чи працездатний процес всередині контейнера. Якщо проба провалюється, Kubernetes знищує та перезапускає (restart) контейнер.
* **Readiness Probe:** перевіряє, чи готовий контейнер приймати мережевий трафік. Якщо проба провалюється, Pod тимчасово виключається зі списку ендпоінтів відповідного Service і на нього перестають надходити запити, але сам контейнер **НЕ перезапускається**.

### c. Через які ресурси проходить запит при зверненні:
* **i. Через `kubectl port-forward`:**
  `Client` - `kubectl (Local Proxy)` - `K8s API Server` - `Pod`.
* **ii. Через `http://k8s-demo.local`:**
  `Client` - `DNS / /etc/hosts` - `Ingress Controller` - `Ingress Resource` - `Service (ClusterIP)` - `Pod`.

### d. Як рекомендується оновлювати конфігурацію, якщо вона надходить через env-змінні з ConfigMap?
Змінні середовища зчитуються процесом застосунку тільки під час старту контейнера. Тому після оновлення ConfigMap необхідно виконати перезапуск Deployment, щоб створити нові Pod-и з оновленими змінними:
```bash
kubectl rollout restart deployment/k8s-demo-api -n k8s-demo
```