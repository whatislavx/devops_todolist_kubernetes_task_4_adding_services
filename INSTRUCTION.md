# Інструкція з тестування застосунку ToDo у Kubernetes

## 1. Передумови
Необхідно налаштувати:
- Kubernetes кластер 
- kubectl
- Docker 

Порт застосунку всередині контейнера: 8000.

## 2. Збірка Docker-образу
Перейдіть у корінь репозиторію, де знаходиться `Dockerfile`.

```cmd
docker build -t todo-app:latest .
```
(За потреби запуште в реєстр замінивши `<REGISTRY>`.)
```cmd
docker tag todo-app:latest <REGISTRY>/todo-app:latest
docker push <REGISTRY>/todo-app:latest
```
```
Застосувати:
```cmd
kubectl apply -f deployment.yaml
```
Перевірити:
```cmd
kubectl get pods -l app=todolist
```

## 3. Сервіс типу ClusterIP
Створіть файл `service-clusterip.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: todo-clusterip
spec:
  type: ClusterIP
  selector:
    app: todolist
  ports:
    - name: http
      port: 8000
      targetPort: 8000
```
Застосувати:
```cmd
kubectl apply -f service-clusterip.yaml
kubectl get svc todo-clusterip
```

## 4. Тест DNS (ClusterIP) з контейнера busybox
Запустити тимчасовий Pod з образом `busybox` і виконати DNS-запити та HTTP-звернення:
```cmd
kubectl run dns-test --image=busybox:1.36 --restart=Never -it --rm -- sh
```
Всередині busybox:
```sh
nslookup todo-clusterip
# Або перевірити повне доменне ім'я (FQDN) якщо є namespace (за замовчуванням 'default'):
nslookup todo-clusterip.default.svc.cluster.local
# Отримати головну сторінку (busybox може не мати wget з SSL підтримкою, але http працює):
wget -qO- http://todo-clusterip:8000/ | head
exit
```
Альтернатива з curl:
```cmd
kubectl run curl-test --image=curlimages/curl:8.7.1 --restart=Never -it --rm -- sh
```
Всередині:
```sh
curl -v http://todo-clusterip:8000/
exit
```

## 5. Тест через port-forward сервісу
Проброс порту сервісу локально:
```cmd
kubectl port-forward svc/todo-clusterip 8000:8000
```
Поки команда активна, у браузері відкрийте:
```
http://localhost:8000/
```
Зупинити CTRL+C.

## 6. Доступ через NodePort
Створіть файл `service-nodeport.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: todo-nodeport
spec:
  type: NodePort
  selector:
    app: todolist
  ports:
    - port: 8000        # порт сервісу
      targetPort: 8000  # порт контейнера
      nodePort: 30080   # бажаний NodePort (повинен бути у діапазоні 30000-32767)
```
Застосувати:
```cmd
kubectl apply -f service-nodeport.yaml
kubectl get svc todo-nodeport
```
Отримати IP вузла (Node):
```cmd
kubectl get nodes -o wide
```
Перевірка доступу (з хоста):
```cmd
curl http://<NODE_IP>:30080/
```
У браузері:
```
http://<NODE_IP>:30080/
```
Якщо використовуєте minikube:
```cmd
minikube ip
curl http://<MINIKUBE_IP>:30080/
```

## 8. Діагностика та пошук проблем
```cmd
kubectl describe deployment todo-deployment
kubectl describe svc todo-clusterip
kubectl logs -l app=todolist --tail=100
kubectl get events --sort-by=.metadata.creationTimestamp
```
Перевірити резолюцію DNS (CoreDNS):
```cmd
kubectl get pods -n kube-system -l k8s-app=kube-dns
```
Якщо сторінка не відкривається:
- Перевірити чи Pod у статусі Running
- Переконатися що порт 8000 відкрито у контейнері
- Перевірити логи на помилки міграцій / імпорту

## 9. Очистка ресурсів
```cmd
kubectl delete svc todo-clusterip todo-nodeport
```
(Переконайтесь що більше не потрібні ці ресурси.)
