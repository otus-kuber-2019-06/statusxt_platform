[![Build Status](https://travis-ci.com/otus-kuber-2019-06/statusxt_platform.svg?branch=master)](https://travis-ci.com/otus-kuber-2019-06/statusxt_platform)

# statusxt_platform
statusxt platform repository

# Table of content
- [Homework-1 kubernetes-intro](#homework-1-kubernetes-intro)
- [Homework-2 kubernetes-security](#homework-2-kubernetes-security)
- [Homework-3 kubernetes-networks](#homework-3-kubernetes-networks)
- [Homework-4 kubernetes-volumes](#homework-4-kubernetes-volumes)

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
apiserver загружается из /etc/kubernetes/manifests и управляется kubelet;
coredns управляется Deployment, который создает соответствующий ReplicaSet;
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
- для того, чтобы файлы, созданные в init контейнере, были доступны основному контейнеру, в pod использован volume типа emptyDir
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

# Homework 2 kubernetes-security
## 2.1 Что было сделано
### 2.1.1 task01
- создан Service Account bob, ему дана роль admin в рамках всего кластера
- создан Service Account dave без доступа к кластеру
```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bob
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bob
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: bob
    namespace: default
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dave
```
### 2.1.2 task02
- создан Namespace prometheus
- создан Service Account carol в этом Namespace
- всем Service Account в Namespace prometheus дана возможность делать get, list, watch в отношении Pods всего кластера
```
---
apiVersion: v1
kind: Namespace
metadata:
  name: prometheus
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: carol
  namespace: prometheus
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-read
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pod-read
subjects:
- kind: Group
  name: system:serviceaccounts:prometheus
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: pod-read
  apiGroup: rbac.authorization.k8s.io
```
### 2.1.3 task03
- создан Namespace dev
- создан Service Account jane в Namespace dev
- jane дана роль admin в рамках Namespace dev
- создан Service Account ken в Namespace dev
- ken дана роль view в рамках Namespace dev
```
---
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jane
  namespace: dev
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ken
  namespace: dev
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin-ns
  namespace: dev
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin-ns-binding
  namespace: dev
subjects:
- kind: ServiceAccount
  name: jane
  namespace: dev
roleRef:
  kind: Role
  name: admin-ns
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: view-ns
  namespace: dev
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: view-ns-binding
  namespace: dev
subjects:
- kind: ServiceAccount
  name: ken
  namespace: dev
roleRef:
  kind: Role
  name: view-ns
  apiGroup: rbac.authorization.k8s.io
```

## 2.2 Как запустить проект
в kubernetes-security:
```
kubectl apply -f ./task01/
kubectl apply -f ./task02/
kubectl apply -f ./task03/
```

## 2.3 Как проверить
```
kubectl get serviceaccounts
kubectl get namespaces
kubectl get serviceaccounts -n dev
kubectl get clusterrole
kubectl get clusterrolebindings 
```

# Homework 3 kubernetes-networks
## 3.1 Что было сделано
- добавлены проверки в описание пода kubernetes-intro/web-pod.yaml
```
      livenessProbe:
        tcpSocket:
          port: 80
      readinessProbe:
        httpGet:
          path: /index.html
          port: 8000
```
- проверка готовности контейнера завершается неудачно
```
 kubectl describe pod/web
```
- создан deployment web-deploy.yaml
- добавьте блок strategy в манифест (web-deploy.yaml)
```
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 100%
```
- опробованы разные варианты деплоя с крайними значениями maxSurge и maxUnavailable (оба 0, оба 100%, 0 и 100%), за процессом можно наблюдать при помощи ()
- создан и применен манифест для сервиса clusterip web-svc-cip.yaml
```
kubectl apply -f web-svc-cip.yaml
kubectl get services
curl http://<CLUSTER-IP>/index.html
ping <CLUSTER-IP>
arp -an
ip addr show
iptables --list -nv -t nat
```
- включен IPVS для kube-proxy - исправлен ConfigMap (конфигурация Pod, хранящаяся в кластере)
```
kubectl --namespace kube-system edit configmap/kube-proxy
kubectl --namespace kube-system delete pod --selector='k8s-app=kube-proxy'
iptables-restore /tmp/iptables.cleanup
```
- установлен MetalLB
```
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.0/manifests/metallb.yaml
kubectl --namespace metallb-system get all
```
- балансировщик настроен с помощью ConfigMap - создан манифест metallb-config.yaml
- создан, применен и проверен манифест web-svc-lb.yaml:
```
kubectl apply -f web-svc-lb.yaml
kubectl get pods -n kube-system
kubectl --namespace metallb-system get all
kubectl describe svc web-svc-lb
curl http://<LB_address>/index.html
```
- установлен ingress-nginx
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
```
- создан и применен файл nginx-lb.yaml c конфигурацией LoadBalancer сервиса
```
kubectl apply -f nginx-lb.yaml
kubectl get service/ingress-nginx
kubectl --namespace ingress-nginx get service/ingress-nginx
curl 172.17.255.2
```
- создан и применен файл web-svc-headless.yaml c конфигурацией Headless сервиса
- настроен ingress-прокси - создан манифест с ресурсом Ingress (web-ingress.yaml)
```
kubectl apply -f web-ingress.yaml
kubectl describe ingress/web
curl http://172.17.255.2/web/index.html
```

## 3.2 Как запустить проект
в kubernetes-networks:
```
kubectl apply -f web-ingress.yaml
```

## 3.3 Как проверить
в kubernetes-networks:
```
kubectl describe ingress/web
```
перейти в браузере:
```
http://<LB_IP>/web/index.html
```

# Homework 4 kubernetes-volumes
## 4.1 Что было сделано
- установлен и запущен kind
```
wget https://github.com/kubernetes-sigs/kind/releases/download/v0.4.0/kind-linux-amd64
chmod +x ./kind-darwin-amd64
mv kind-linux-amd64 /usr/local/bin/kind
kind create cluster
export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"
kubectl cluster-info
```
- создана и применена конфигурация statefulset minio
- для того, чтобы	StatefulSet был доступен изнутри кластера, создан Headless Service minio-headlessservice.yaml
### 4.1.1 В рамках задания со (*)
- в конфигурацию statefulset добалено использование secrets:
```
        env:
        - name: MINIO_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: minio-secrets
              key: MINIO_ACCESS_KEY
        - name: MINIO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: minio-secrets
              key: MINIO_SECRET_KEY
```
- создан манифест secrets minio-secrets.yaml
```
echo -n 'minio' | base64
echo -n 'minio123' | base64
kubectl apply -f minio-secrets.yaml
kubectl apply -f minio-statefulset.yaml
get secret minio-secrets -o yaml
```

## 4.2 Как запустить проект
в kubernetes-volumes:
```
kubectl apply -f minio-statefulset.yaml
kubectl apply -f minio-headless-service.yaml
```

## 4.3 Как проверить
```
kubectl get pods
kubectl get statefulsets
kubectl get pv
kubectl get pvc
kubectl get services
```
