# Інструкція з тестування застосунку ToDo у Kubernetes (оновлено під актуальні порти)

## 1. Передумови
Необхідно налаштувати:
- Kubernetes кластер 
- kubectl
- Docker 

Порт застосунку всередині контейнера: 8080 (targetPort). Сервіс надає доступ по порту 80 (port). NodePort відкриває порт 30080 на вузлі.

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

```
## 3. Сервіс типу ClusterIP
Створіть файл `service-clusterip.yaml` (оновлені порти):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: clusterip            # фактичне ім'я сервісу; замініть якщо у вашому YAML інше
spec:
  type: ClusterIP
  selector:
    app: todolist
  ports:
    - name: http
      port: 80               # порт сервісу (client-facing)
      targetPort: 8080       # порт контейнера
```
Застосувати:
```cmd
kubectl apply -f service-clusterip.yaml
kubectl get svc clusterip
```
(Якщо ваш сервіс має іншу назву, адаптуйте команди; раніше приклади з `todo-clusterip` були замінені.)

## 4. Тест DNS (ClusterIP) з контейнера busybox
```cmd
kubectl run dns-test --image=busybox:1.36 --restart=Never -it --rm -- sh
```
Всередині busybox:
```sh
nslookup clusterip
nslookup clusterip.default.svc.cluster.local
wget -qO- http://clusterip:80/ | head
exit
```
Альтернатива з curl:
```cmd
kubectl run curl-test --image=curlimages/curl:8.7.1 --restart=Never -it --rm -- sh
```
Всередині:
```sh
curl -v http://clusterip:80/
exit
```

## 5. Тест через port-forward сервісу
Проброс порту сервісу локально (локальний порт 8080 -> сервісний 80):
```cmd
kubectl port-forward svc/clusterip 8080:80
```
У браузері:
```
http://localhost:8080/
```
Зупинити CTRL+C.

## 6. Доступ через NodePort
Створіть файл `service-nodeport.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport
spec:
  type: NodePort
  selector:
    app: todolist
  ports:
    - port: 80          # порт сервісу
      targetPort: 8080  # порт контейнера
      nodePort: 30080   # зовнішній порт на вузлі
```
Застосувати:
```cmd
kubectl apply -f service-nodeport.yaml
kubectl get svc nodeport
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
Для minikube:
```cmd
minikube ip
curl http://<MINIKUBE_IP>:30080/
```

## 8. Діагностика та пошук проблем
Опис ресурсів:
```cmd
kubectl describe svc clusterip
kubectl describe svc nodeport
```
Перевірка Pod (наприклад ваш `todoapp-pod1`):
```cmd
kubectl describe pod todoapp-pod1
kubectl logs todoapp-pod1 --tail=100
```
Якщо використовуєте Deployment:
```cmd
kubectl describe deployment todo-deployment
```
Події:
```cmd
kubectl get events --sort-by=.metadata.creationTimestamp
```
DNS (CoreDNS):
```cmd
kubectl get pods -n kube-system -l k8s-app=kube-dns
```
Якщо сторінка не відкривається:
- Pod не в статусі Running
- Невірний порт у сервісі (має бути port:80 targetPort:8080)
- Застосунок слухає інший порт (перевірити logs)
- Проблеми міграцій / виключення у Django

## 8. Очистка ресурсів
```cmd
kubectl delete svc clusterip nodeport
# або видалити окремий Pod якщо вручну створювали:
# kubectl delete pod todoapp-pod1
```

## 10. Port mapping резюме
- Всередині контейнера: 8080 
- ClusterIP service: port 80 -> targetPort 8080
- NodePort service: nodePort 30080 -> port 80 -> targetPort 8080
- Локальний port-forward: localhost:8080 -> svc/clusterip:80