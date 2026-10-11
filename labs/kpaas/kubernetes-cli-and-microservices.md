---
Date: 2026-01-05
tags:
  - type/practice
  - topic/cloud
aliases:
  - kubernetes-cli
  - namespace
  - service
  - deployment
  - pod
---

# Kubernetes CLI and Microservices Deployment

## Kubernetes CLI Lab
### 1. Deploying and Inspecting Namespaces
```sh
# kpaas 네임스페이스 배포
$ kubectl create ns kpaas

# namespace 목록
$ kubectl get namespace

# namespace 상세 조회
$ kubectl describe ns kpaas
```
<img width="1741" height="634" alt="Pasted image 20260105104832" src="https://github.com/user-attachments/assets/3daae8bf-9a02-4e83-95fb-04571f90954c" />
<img width="1489" height="321" alt="Pasted image 20260105104957" src="https://github.com/user-attachments/assets/e12ae6d8-3924-49fb-afbc-8f1ff74289f5" />

###### YAML: Namespace Deployment
```yaml
# ns-kpaas.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kpaas
  labels:
    name: kpaas
    environment: dev # Optional
```
```sh
$ kubectl apply -f ns-kpaas.yaml
```
<img width="626" height="435" alt="Pasted image 20260113152025" src="https://github.com/user-attachments/assets/afbe0ba4-2e34-4ee2-b212-4aa194bc590e" />

### 2. Deploying and Inspecting Pods
```sh
# kpaas-nginx Pod 배포
# nginx:1.25 컨테이너 이미지를 kpaas namespace에 격리하여 배포
$ kubectl run kpaas-nginx --image=nginx:1.25 -n kpaas

# Pod 목록 조회 
$ kubectl get pod -n kpaas

# Pod 상세 조회
$ kubectl describe pod kpaas-nginx -n kpaas
```
<img width="2306" height="1087" alt="Pasted image 20260105105308" src="https://github.com/user-attachments/assets/cc623375-aee4-4a59-a4ae-6fa89ed3a16c" />

###### YAML: Pod Deployment
```yaml
# nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kpaas-nginx       # Pod 이름
  namespace: kpaas        # 네임스페이스 지정 (-n kpaas)
spec:
  containers:
  - name: kpaas-nginx     # 컨테이너 이름
    image: nginx:1.25     # 이미지 태그 지정 (--image=nginx:1.25)
    ports:
    - containerPort: 80   # Nginx 기본 포트
```
```sh
$ kubectl apply -f nginx-pod.yaml
```
<img width="1673" height="454" alt="Pasted image 20260113152554" src="https://github.com/user-attachments/assets/c8d6524d-7088-442c-a931-a26f40e0b382" />

### 3. Service Deployment
```sh
# Service 배포 (Pod 연결)
$ kubectl expose pod kpaas-nginx --type=NodePort --port 80 --target-port=80 --name=kpaas-nginx -n kpaas

# Service 조회
$ kubectl get service -n kpaas

# Service 상세 조회
$ kubectl describe service kpaas-nginx -n kpaas
```
- `--port 80`: Service Port
	- 클라이언트, 클러스터 내부의 다른 Pod가 Service에 접근할 때 사용하는 Port
- `--target-port=80`: Container Port
	- 실행 중인 Pod 내부의 애플리케이션 Port
	- Service가 받은 요청을 실제 애플리케이션으로 Forwarding 해줄 때 사용
	- **컨테이너 이미지에 정의된 포트와 일치해야 함.**
```
외부/클라이언트  ──▶  [ Service ]  ──▶  [ Pod (Container) ]
                      │                 │
                  --port (80)       --target-port (80)
```
<img width="2132" height="777" alt="Pasted image 20260105105644" src="https://github.com/user-attachments/assets/27714e62-d3d1-4a5b-815a-6bd3bf94b55c" />

###### YAML: Service Deployment
```yaml
# nginx-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: kpaas-nginx      # 서비스 이름 (--name)
  namespace: kpaas       # 네임스페이스 (-n)
spec:
  type: NodePort         # 서비스 타입 (--type=NodePort)
  selector:
    run: kpaas-nginx     # 연결할 Pod와 일치해야 함
  ports:
  - port: 80             # Service 접속 포트 (--port)
    targetPort: 80       # Pod 내부 컨테이너 포트 (--target-port)
    protocol: TCP
```
- `kubectl run`으로 만든 Pod는 기본적으로 `run` label을 가지므로 service를 붙일 때 참고
	- `selector: run: kpaas-nginx` (`run=kpaas-nginx`)
- Service는 `selector`에 해당하는 label을 가진 Pod를 가리킴
```sh
$ kubectl apply -f nginx-pod.yaml
```
<img width="1279" height="531" alt="Pasted image 20260113152936" src="https://github.com/user-attachments/assets/fe236ba8-a102-4e52-98ff-4d518a67a58c" />

### 4. Deployment Creation
#### 4-1. Creating a Deployment
```sh
# Deployment 배포
$ kubectl create deployment kpaas-nginx2 --image=nginx:1.26 --port=80 --replicas=2 -n kpaas

# Deployment 목록 조회
$ kubectl get deploy -n kpaas

# Deployment 상세 조회
$ kubectl describe pod kpaas-nginx2 -n kpaas
```
- `--port=80`: Container Port
	- 내부 컨테이너가 사용하는 Port
- `--replicas=2`
	- Pod 2개 유지
	- 서버 하나가 죽어도 나머지 하나가 살아있어서 무중단 서비스 가능
	- `kpaas-nginx2` Pod 2개 생성
<img width="2345" height="945" alt="Pasted image 20260105110403" src="https://github.com/user-attachments/assets/587c4335-101e-4b0e-b9b9-600fdb7e8bac" />

###### YAML: Deployment Creation
```yaml
# nginx2-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kpaas-nginx2      # Deployment 이름
  namespace: kpaas        # 네임스페이스 (-n kpaas)
  labels:
    app: kpaas-nginx2     # Deployment 자체의 label
spec:
  replicas: 2             # 파드 개수 (--replicas=2)
  selector:
    matchLabels:
      app: kpaas-nginx2   # 관리할 Pod의 label
  template:
    metadata:
      labels:
        app: kpaas-nginx2 # 생성될 Pod에 붙을 label
    spec:
      containers:
      - name: nginx       # 컨테이너 이름 
        image: nginx:1.26 # 이미지 태그 (--image=nginx:1.26)
        ports:
        - containerPort: 80 # 포트 설정 (--port=80)
```
```sh
$ kubectl apply -f nginx2-deploy.yaml
$ kubectl describe pod kpaas-nginx2 -n kpaas
```
<img width="1258" height="472" alt="Pasted image 20260113153816" src="https://github.com/user-attachments/assets/62db2dc5-a9c4-4f75-8492-9e8d0a6d67b5" />

#### 4-2. Connecting a Service to a Deployment
```sh
# Service 배포 (Deployment 연결)
$ kubectl expose deploy kpaas-nginx2 --type=NodePort --port=80 --target-port=80 --name=kpaas-nginx2 -n kpaas

# Service 목록 조회
$ kubectl get service -n kpaas

# Service 상세 조회
$ kubectl describe service kpaas-nginx -n kpaas
```
<img width="2245" height="829" alt="Pasted image 20260105110835" src="https://github.com/user-attachments/assets/f5494325-4480-4f55-9f17-1ec40e3fdf71" />

###### yaml:
```yaml
# nginx2-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: kpaas-nginx2     # 서비스 이름 (--name)
  namespace: kpaas       # 네임스페이스 (-n)
spec:
  type: NodePort         # 서비스 타입 (--type=NodePort)
  selector:
    app: kpaas-nginx2    # Deployment의 label과 일치해야 함
  ports:
  - port: 80             # Service Port (--port)
    targetPort: 80       # Container Port (--target-port)
    protocol: TCP
```
- `kubectl create deployment`로 만든 deployment는 기본적으로 `app` label을 가짐
	- 따라서 Service가 Deployment가 관리하는 Pod를 찾으려면 `app` label을 가리켜야 함
	- `selector: app: kpaas-nginx2`
```sh
$ kubectl apply -f nginx2-svc.yaml
$ kubectl describe svc kpaas-nginx2 -n kpaas
```
<img width="928" height="381" alt="Pasted image 20260113154037" src="https://github.com/user-attachments/assets/cacc84f8-17a3-4cb5-a8b2-0823b5493a3f" />

#### 4-3. Updating Deployment Configuration
```sh
# Deployment의 replicas 설정 변경
$ kubectl edit deploy kpaas-nginx2 -n kpaas

# replicas 2에서 4로 변경 후 
# Pod 목록 조회
$ kubectl get pod -n kpaas
# Pod 개수 4개로 늘어남
```
<img width="1906" height="488" alt="Pasted image 20260105111306" src="https://github.com/user-attachments/assets/9c0011bc-18f2-48fa-95f8-819f0d6da703" />
<img width="1497" height="624" alt="Pasted image 20260105111521" src="https://github.com/user-attachments/assets/08aed928-9853-4d48-bf0b-954ec29798ce" />

### 5. Resetting Resources After the Lab
```sh
$ kubectl delete -f .
pod "kpaas-nginx" deleted
service "kpaas-nginx" deleted
deployment.apps "kpaas-nginx2" deleted
service "kpaas-nginx2" deleted
namespace "kpaas" deleted
```
<img width="706" height="92" alt="Pasted image 20260113154642" src="https://github.com/user-attachments/assets/4c42b85a-1525-4b0e-b846-421f2148ed9d" />

## Building a Microservices Application with Container Platform Services
- MSA Sample 가이드 주소: https://github.com/K-PaaS/msa-sample
<img width="2081" height="961" alt="Pasted image 20260112163651" src="https://github.com/user-attachments/assets/f206b177-6985-41d2-a93c-7ca482efefd2" />

#### Architecture
<img width="2375" height="1140" alt="Pasted image 20260112163750" src="https://github.com/user-attachments/assets/c869ef9e-5cdd-43b8-9308-0ffc9e3634bf" />

##### 1. msa-sample-ui
- 화면 서비스를 담당하는 API
##### 2. msa-sample-gateway
- API 서버 앞단에서 여러 API 서버들의 Endpoint를 단일화하여 
- 라우팅, 로드밸런싱 등의 역할을 하는 API
##### 3. msa-sample-api & msa-sample-sd-api
- 각각 메인 화면/side 화면 정보에 대한 서비스를 담당하는 api
### 0. unseal `openbao`
```sh
$ kubectl get pod -n openbao
# 초기: openbao READY 0/1
```
<img width="1503" height="173" alt="Pasted image 20260105103801" src="https://github.com/user-attachments/assets/299aa327-8fe3-45ad-b314-5596d3257ecb" />

```sh
#1. 배포된 Vault의 Unseal Key 확인
$ cat ~/workspace/container-platform/cp-portal-deployment/secmg/unseal-key

#2. 'openbao-0' pod에 접속 
$ kubectl exec -it openbao-0 -n openbao -- sh

#3. vault unseal 상태로 변경하기
##3-1. unseal 첫번째 키 입력
/ $ vault operator unseal
Unseal Key (will be hidden):  💠# 첫번째 키 입력

##3-2. unseal 두번째 키 입력
/ $ vault operator unseal
Unseal Key (will be hidden):  💠# 두번째 키 입력

/ $ exit

#4. 정상 작동 확인
$ kubectl get pod -n openbao
NAME                                      READY   STATUS    RESTARTS   AGE
openbao-0                                 1/1     Running   5          6d18h
openbao-agent-injector-6567764cc9-7zx8l   1/1     Running   3          4d20h
```
<img width="2018" height="1278" alt="Pasted image 20260105112402" src="https://github.com/user-attachments/assets/ecffccaa-fcbc-42b3-914f-321b5f6da548" />

###### `503` Error When `openbao` Is Not Working
- `https://portal.133.186.xxx.xxx.nip.io/`
<img width="895" height="356" alt="Pasted image 20260105104624" src="https://github.com/user-attachments/assets/59cb9cdc-4138-4cf9-9d6d-ea0d3ed6c49f" />
<img width="917" height="172" alt="Pasted image 20260105110428" src="https://github.com/user-attachments/assets/db465d7c-9976-4eab-816e-40fd7ce2ec99" />

### 1. Environment Variable Configuration
#### 1-1. Portal User Login
```sh
$ kubectl get ingress -A
# portal.133.186.xxx.xxx.nip.io 접속

# Username: admin
# Password: 
```
#### 1-2. Creating a `ConfigMap`
- [ConfigMap YAML 링크](https://github.com/K-PaaS/msa-sample/blob/master/resource/configmap.yml)
- `{K8S_MASTER_NODE_IP}`에 Control Plane Node의 Public IP 주소 입력
```sh
apiVersion: v1
kind: ConfigMap
metadata:
  name: kpaas-msa-configmap
  namespace: default
data:
  MSA_API_URL: http://133.186.xxx.xxx:30010
  MSA_SD_API_URL: http://133.186.xxx.xxx:30014
  MSA_UI_URL: http://133.186.xxx.xxx:30011
  MSA_GATEWAY_URL: http://133.186.xxx.xxx:30012
```
<img width="1061" height="694" alt="Pasted image 20260105114052" src="https://github.com/user-attachments/assets/48bfc62c-3430-4402-ba15-8dfb1fc0fcfb" />

### 2. Repository Creation
#### 2-1. Creating an Empty Repository in the K-PaaS SCM Portal
<img width="1045" height="802" alt="Pasted image 20260105114837" src="https://github.com/user-attachments/assets/a44fb0a3-1168-41cf-b25f-0ea4918862b8" />

#### 2-2. `git clone` 
- `cluster1`에서 Github 소스코드 다운로드
	- `--mirror`: Commit history, branch, tag 등 관리 이력만 담고 있는 형태로 가져옴.
	- 저장소 자체를 이동시키기 위함
<img width="1758" height="1092" alt="Pasted image 20260105114612" src="https://github.com/user-attachments/assets/b3b12096-3385-446d-9539-e3368ebe38c8" />

#### 2-3. Pushing to the SCM Server
- 다운로드한 디렉토리 내부로 이동 후
- 내부 SCM 서버로 데이터를 전송한다.
	- `git push --mirror [SCM_URL]`
	- 로컬에 가지고 있던 모든 Git 이력을 내부 서버의 빈 저장소로 Push
<img width="2264" height="678" alt="Pasted image 20260105115031" src="https://github.com/user-attachments/assets/7660b7af-4d07-4401-8c5d-664d93ec263f" />
<img width="978" height="704" alt="Pasted image 20260105115048" src="https://github.com/user-attachments/assets/0816937a-4715-4b09-ad34-db909f75715d" />
<img width="2299" height="522" alt="Pasted image 20260105115224" src="https://github.com/user-attachments/assets/27eb7d5d-a167-438c-a3e4-42fa7fbbd2b1" />
<img width="959" height="775" alt="Pasted image 20260105115235" src="https://github.com/user-attachments/assets/49216bfd-efed-490f-8472-d8220bcba364" />
<img width="2220" height="499" alt="Pasted image 20260105115330" src="https://github.com/user-attachments/assets/0e2e29cf-3fdc-4878-b574-bd66e6caf509" />
<img width="847" height="775" alt="Pasted image 20260105115356" src="https://github.com/user-attachments/assets/0299f06a-bb85-4384-8f8e-d6e93133972e" />
<img width="2331" height="519" alt="Pasted image 20260105115442" src="https://github.com/user-attachments/assets/12ec862f-812b-479d-a8f2-e1039d256778" />
<img width="933" height="769" alt="Pasted image 20260105115503" src="https://github.com/user-attachments/assets/d9525e1a-145e-43d7-931b-1d6d4903af14" />

### 3. Pipeline Configuration
#### 3-1. Retrieving and Registering Kube Config
```sh
$ ls .kube
$ cat .kube/kubeconfig # config 조회
```

```sh
# kubeconfig

apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: 
	    ...
    server: https://133.186.xxx.xxx:6443
  name: cluster-host
contexts:
- context:
    cluster: cluster-host
    user: cluster-host-admin
  name: cluster-host
current-context: cluster-host
kind: Config
preferences: {}
users:
- name: cluster-host-admin
  user:
    client-certificate-data: 
	    ...
    client-key-data: 
	    ...
```
- 파이프라인 관리 - Kubernetes 정보 관리 - Kube Config 등록
<img width="755" height="296" alt="Pasted image 20260105121303" src="https://github.com/user-attachments/assets/534d1421-5544-46f2-87ff-e021df1117ac" />

### 4. Pipeline Creation
<img width="919" height="176" alt="Pasted image 20260105121410" src="https://github.com/user-attachments/assets/4cb10e0c-a0c1-41e9-97ed-5a88b7cee6a1" />
<img width="853" height="801" alt="Pasted image 20260105121649" src="https://github.com/user-attachments/assets/0c8f0845-95f6-4e53-947e-9a3c067d4014" />
- `openjdk` $\rightarrow$ `amazoncorretto` 이미지 변경
<img width="919" height="642" alt="Pasted image 20260105121808" src="https://github.com/user-attachments/assets/19a54980-2d3d-4ffc-b6d9-8a62190754e5" />
<img width="1006" height="598" alt="Pasted image 20260105122018" src="https://github.com/user-attachments/assets/d637726c-9cfd-412c-80b0-0993b9344bc8" />
<img width="980" height="607" alt="Pasted image 20260105122158" src="https://github.com/user-attachments/assets/94cad45e-d2ed-4df2-9236-f17750bd4016" />
<img width="1051" height="406" alt="Pasted image 20260105122249" src="https://github.com/user-attachments/assets/8ba615ac-f761-42d0-833b-3f453ef1d7f3" />

<img width="1047" height="818" alt="Pasted image 20260105124655" src="https://github.com/user-attachments/assets/de2121f6-f6ba-4246-893d-7fac13e792e8" />
<img width="1072" height="915" alt="Pasted image 20260105124755" src="https://github.com/user-attachments/assets/69fa233b-2f24-4b34-8313-1bc3d2e4118b" />
