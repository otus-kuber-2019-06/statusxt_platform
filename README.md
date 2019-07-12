[![Build Status](https://travis-ci.com/otus-kuber-2019-06/statusxt_platform.svg?branch=master)](https://travis-ci.com/otus-kuber-2019-06/statusxt_platform)

# statusxt_platform
statusxt platform repository

# Table of content
- [Homework-1 kubernetes-intro](#homework-1-kubernetes-intro)


# Homework 1 kubernetes-intro
## 1.1 Что было сделано
- установлен docker-ce
- установлен kubectl
- установлен и запущен minikube:
```
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \\
  && chmod +x minikube
sudo install minikube /usr/local/bin
minikube start --vm-driver=none
```
- проверено подключение к кластеру:
```
kubectl cluster-info
```
- установлены dashboard и k9s
```
minikube dashboard
snap install k9s
```
- протестирована отказоустойчивость k8s:
```
docker rm -f $(docker ps -a -q)
kubectl get pods -n kube-system
kubectl delete pod --all -n kube-system
kubectl get componentstatuses
```
- механизмы обеспечения отказоустойчивости различны:
apiserver загружается из /etc/kubernetes/manifests и управляется kubelet
coredns управляется Deployment, который создает соответствующий ReplicaSet
kube-proxy управляется DaemonSet
- создан образ контейнера с простейшим веб-сервером и загружен на DockerHub:
```
docker build -t statusxt\custom-nginx .
docker tag custom-nginx:latest statusxt/web
docker push statusxt/web
```
- описан манифест web-pod.yaml для создания pod web c меткой app со значением web, содержащий один контейнер сназванием web. Использован ранее собранный образ с DockerHub.
- в манифесте указан несуществующий тег образа web и применен заново, статус пода изменился:
```
kubectl apply -f web-pod.yaml
kubectl get pods
kubectl describe pod
```
- в pod добавлен init контейнер
- для того, чтобы файлы, созданные в init контейнере, были доступны основному контейнеру в pod использован volume типа emptyDir
- проверена работоспособность приложения
```
kubectl delete pod web
kubectl apply -f web-pod.yaml && kubectl get pods -w
kubectl port-forward --address 0.0.0.0 pod/web 8000:8000
```

## 1.2 Как запустить проект
в kubernetes-intro:
```
kubectl delete pod web
kubectl apply -f web-pod.yaml
```

## 1.3 Как проверить
в kubernetes-intro:
```
kubectl port-forward --address 0.0.0.0 pod/web 8000:8000
curl http://localhost:8000/index.html
```

