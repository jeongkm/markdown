# Secret 

* 용도 : 패스워드나 API 키 등 기밀 정보를 저장하고, 같은 네임스페이스에서 실행되고 있는 컨테이너나 배포 (Pod+ReplicaSet)에서 마운트해 사용

* [Managing Secret](https://www.ibm.com/support/knowledgecenter/ko/SSBS6K_3.1.1/manage_applications/create_secrets.html#encrypt)



## Secret 생성

1. kubectl CLI를 Secret을 생성할 네임스페이스로 스위치

   ```
   kubectl config set-context <cluster_CA_domain>-context --user=<user_name> --namespace=<namespace_name>
   ```

2. kubectl 명령어로 생성하길 원하는 Secret 유형을 정해 실행

   * kubectl create secret docker-registry : imagePullSecretes 생성
     * [Creating imagePullSecrets for a specific namespace](https://www.ibm.com/support/knowledgecenter/ko/SSBS6K_3.1.1/manage_images/imagepullsecret.html?view=kc) and the [kubectl create secret docker-registry command ![Opens in a new tab](https://www.ibm.com/support/knowledgecenter/ko/SSBS6K_3.1.1/images/icons/launch-glyph.svg)](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-secret-docker-registry-em-).
   * kubectl create secret generic : 로컬 파일이나 디렉토리로부터 생성
     *  [kubectl create secret generic command ![Opens in a new tab](https://www.ibm.com/support/knowledgecenter/ko/SSBS6K_3.1.1/images/icons/launch-glyph.svg)](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-secret-generic-em-)
   * kubectl create secret tls : public/private key pair로부터 생성
     * [kubectl create secret tls command ![Opens in a new tab](https://www.ibm.com/support/knowledgecenter/ko/SSBS6K_3.1.1/images/icons/launch-glyph.svg)](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-secret-tls-em-)



### kubectl 명령어로 생성

```
# kubectl create secret generic user-password --from-literal=user=admin --from-literal=password=passw0rd
secret/user-password created

# kubectl describe secrets/user-password
Name:         user-password
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  8 bytes
user:      5 bytes
```



### yaml 파일로 생성

base64 인코딩된 아이디와 패스워드 준비

```
# echo -n "admin" | base64
YWRtaW4=

# echo -n "passw0rd" | base64
cGFzc3cwcmQ=
```



secret.yaml

```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: cGFzc3cwcmQ=
```



생성

```
# kubectl apply -f secret.yaml
secret "mysecret" created
```



## Secret 확인

```
# kubectl get secret mysecret -o yaml
# kubectl get secret mysecret -o yaml

apiVersion: v1
data:
  password: cGFzc3cwcmQ=
  username: YWRtaW4=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"password":"cGFzc3cwcmQ=","username":"YWRtaW4="},"kind":"Secret","metadata":{"annotations":{},"name":"mysecret","namespace":"kube-system"},"type":"Opaque"}
  creationTimestamp: 2019-01-13T03:47:31Z
  name: mysecret
  namespace: kube-system
  resourceVersion: "447561"
  selfLink: /api/v1/namespaces/kube-system/secrets/mysecret
  uid: f471e5f4-16e5-11e9-9f9d-06357399dad6
type: Opaque
```



### Secret 디코딩

```
# echo "YWRtaW4=" | base64 --decode
admin

# echo "cGFzc3cwcmQ=" | base64 --decode
passw0rd
```



## Secret 사용

### Secret 을 Pod에 환경변수로 삽입



gs-spring-boot-deployment.yaml

```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: spring-boot-deployment
  labels:
    app: spring-boot-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-boot-app
  template:
    metadata:
      labels:
        app: spring-boot-app
    spec:
      containers:
      - name: spring-boot-app
        image: springio/gs-spring-boot-docker:latest
        ports:
        - containerPort: 8080
        env:
        - name: SECRET_USERNAME
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: username
        - name: SECRET_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: password
      restartPolicy: Always
```



deployment 배포 및 pod 확인

```
# kubectl apply -f gs-spring-boot-deployment.yaml 
deployment.apps/spring-boot-deployment created

# kubectl get pod | grep spring
spring-boot-deployment-6c7547c8d9-r7s9j                           1/1       Running            0          41s

```



환경변수 확인

```
# kubectl exec -it spring-boot-deployment-6c7547c8d9-r7s9j sh
/ # env | grep SECRET
SECRET_PASSWORD=passw0rd
SECRET_USERNAME=admin
/ # 
```



## Secret 변경



base64 인코딩된 새 패스워드로 변경 후, 저장

```
# echo "passw0rd!@" | base64
cGFzc3cwcmQhQAo=

# kubectl edit secret mysecret

      7   password: cGFzc3cwcmQhQAo=
      8   username: YWRtaW4=
```



수정된 Secret을 적용하기 위해 Pod를 재생성

```
# kubectl get pod -l app=spring-boot-app -o yaml | kubectl replace --force -f -
pod "spring-boot-deployment-6c7547c8d9-r7s9j" deleted
pod/spring-boot-deployment-6c7547c8d9-r7s9j replaced 
```



업데이트된 password 환경변수 확인

```
# kubectl get pod | grep spring
spring-boot-deployment-6c7547c8d9-fgb9k                           1/1       Running            0          2m

# kubectl exec -it spring-boot-deployment-6c7547c8d9-fgb9k sh
/ # env | grep SECRET
SECRET_PASSWORD=passw0rd!@
SECRET_USERNAME=admin
```



## Secrets 암호화



Secret은 etcd 스토리지에 base64 인코딩된 형태로 저장됨.

kube-apiserver에 --encryption-provider-config 옵션 적용 시, etcd에 암호화된 secret이 저장됨.

k8s v1.13.0 부터 정식 지원, 이전 버전에서는 --experimental-encryption-provider-config 로 알파 버전 제공.

Kubernetes doc : [Encrypting Secret Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)



### Encryption 설정

1. 32바이트 랜덤키 생성해 base64 인코딩

   ```
   # head -c 32 /dev/urandom | base64
   3djYtAK9O+7pBbo5oAFKMJAqj4GQS10IhXEXYeMbRKw=
   ```

2. encryption-config.yaml에 랜덤키를 기입하고, 모든 마스터 노드의 /etc/cfc/conf 폴더에 복사

   ```
   kind: EncryptionConfig
   apiVersion: v1
   resources:
    - resources:
      - secrets
      providers:
      - aescbc:
          keys:
          - name: key1
            secret: <base64-encoded Secret>
      - identity: {}
   ```

   

   ICP 클러스터에는 이미 파일이 존재함

   ```
   # cat /etc/cfc/conf/encryption-config.yaml
   kind: EncryptionConfig
   apiVersion: v1
   resources:
     - resources:
       - secrets
       providers:
       - aescbc:
           keys:
           - name: key1
             secret: btGbAa/2BPaPlc1zVt4x6KfwVj+KTAxBipuVa3Mo7Zw=
       - kms:
             name: KmsPlugin
             endpoint: unix:///tmp/keyprotectprovider.sock
             cachesize: 100
       - identity: {}
   ```

   

3. 모든 마스터 노드의 /etc/cfc/pods/master.json 에 encryption-config.yaml 파일 위치 지정

   ```
   // 백업
   # cp /etc/cfc/pods/master.json ~/master.json.bak
   
   // 복사
   # cp /etc/cfc/pods/master.json /tmp
   
   // 편집
   // 157라인에 "--experimental-encryption-provider-config=/etc/cfc/conf/encryption-config.yaml 추가"
   # vi /tmp/master.json
   
       155           "--profiling=false",
       156           "--service-cluster-ip-range=10.0.0.0/16",
       157           "--experimental-encryption-provider-config=/etc/cfc/conf/encryption-config.yaml"
       158         ],
       159         "volumeMounts": [
       
   // Overwrite
   # cp /tmp/master.json /etc/cfc/pods/
   cp: overwrite `/etc/cfc/pods/master.json'? y
   
   // apiserver 재시작 확인
   # docker ps | grep apiserver
   60bb5be2cfd9        4c7c25836910                              "/hyperkube apiserve…"   4 seconds ago       Up 4 seconds                            k8s_apiserver_k8s-master-169.56.108.4_kube-system_a48cd4963ad9c3c33befcae4a028fbe0_0
   ec29aae25927        c3491fd85080                              "/opt/services/servi…"   ...
   
   ```



### 암호화 대상 Secret 확인

암호화 적용 대상 확인

```
# kubectl -n kube-system get secret platform-auth-idp-credentials -o yaml
apiVersion: v1
data:
  admin_password: YWRtaW4=
  admin_username: YWRtaW4=
kind: Secret
metadata:
  creationTimestamp: 2019-01-11T05:50:45Z
  name: platform-auth-idp-credentials
  namespace: kube-system
  resourceVersion: "3003"
  selfLink: /api/v1/namespaces/kube-system/secrets/platform-auth-idp-credentials
  uid: d6eb8612-1564-11e9-b060-06357399dad6
type: Opaque
```



etcdctl CLI 설정 : etcd 로부터 secret을 읽기 위해 사용

```
# docker ps | grep k8s_etcd_k8s-etcd
a35dd72133ca        33bdcac177c2                              "etcd --name=etcd0 -…"   2 days ago           Up 2 days                               k8s_etcd_k8s-etcd-169.56.108.4_kube-system_8e834897e56389eb13b9abc8df3b1d64_0

# docker cp a35dd72133ca:/usr/local/bin/etcdctl /usr/local/bin/

# endpoint = 169.56.108.4
# alias etcdctl3="ETCDCTL_API=3 etcdctl --endpoints=$endpoint:4001 --cacert=/etc/cfc/conf/etcd/ca.pem --cert=/etc/cfc/conf/etcd/client.pem --key=/etc/cfc/conf/etcd/client-key.pem"
```



etcd에 저장되어 있는 secret 정보 읽기

```
# etcdctl3 get -w fields /registry/secrets/kube-system/platform-auth-idp-credentials
"ClusterID" : 18198080107246212393
"MemberID" : 11619985723144134093
"Revision" : 464432
"RaftTerm" : 2
"Key" : "/registry/secrets/kube-system/platform-auth-idp-credentials"
"CreateRevision" : 3003
"ModRevision" : 3003
"Version" : 1
"Value" : "k8s\x00\n\f\n\x02v1\x12\x06Secret\x12\xa2\x01\nf\n\x1dplatform-auth-idp-credentials\x12\x00\x1a\vkube-system\"\x00*$d6eb8612-1564-11e9-b060-06357399dad62\x008\x00B\b\b\xb5\xdd\xe0\xe1\x05\x10\x00z\x00\x12\x17\n\x0eadmin_password\x12\x05admin\x12\x17\n\x0eadmin_username\x12\x05admin\x1a\x06Opaque\x1a\x00\"\x00"
"Lease" : 0
"More" : false
"Count" : 1

```



### etcd 내 모든 Secret  암호화

백업

```
kubectl get secrets --all-namespaces -o json > mysecrets.json
```



암호화 적용

```
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```



암호화 적용 확인

```
# etcdctl3 get -w fields /registry/secrets/kube-system/platform-auth-idp-credentials
"ClusterID" : 18198080107246212393
"MemberID" : 11619985723144134093
"Revision" : 464790
"RaftTerm" : 2
"Key" : "/registry/secrets/kube-system/platform-auth-idp-credentials"
"CreateRevision" : 3003
"ModRevision" : 464709
"Version" : 2
"Value" : "k8s:enc:aescbc:v1:key1:k\xcf\xc3\xf0(\f:\xa9da\xb9\xf8\x19\u007f\vs#\xcf6\xbc\u0092\xfd?\x92לW\xc9\x16>g\x9f\xea\xdf&\xec\x94m\xec\x14\x99\xa7e\x90$\xc2O\xf5\xd0\xf8\xa6\x94X\xfd%O_\xca\xe5?]\xc49\xc36\xf4\xfd:x\xdd0\\{\xb8ps\xe79^\x8e߀\xa4\r>ŗ\xc0?\x99\x99\xb7j\x91\xdb\xdc`\x0f\xc0\x8b6\x05N\xc7Q\xcb\xed\xdbٴ\xf2\x10\a\x11A\xe3b\x88\xb6\x9e\xb1\xdf\xdd\xcbN\x11\xaa\x03/7\xcdU]\x12\x0e\xb7\xe4\xe88\\\xc4һ\x1a\x97n\xb2ja\x18\x88\xd1\x0e\x12\x1a\xc7\x1e\xb7G\x9b?W\xc8fPův\xe3\x97q\xd09\xcd?\x86\xa4ܼ\x12\xff\xb93B6\x03\xe1kP\xe4VA\xbf\x14<o\xcd&2\xf1ĳ\xdcf\x14,="
"Lease" : 0
"More" : false
"Count" : 1
```



### 복호화 키 변경



새 키 생성

```
# head -c 32 /dev/urandom | base64
yXNoEPj4Rv/S/AhctTCLd0ooTN/F1rPk1DPU4gFyY28=
```



키 등록 

```
# vi encryption-config.yaml

      1 kind: EncryptionConfig
      2 apiVersion: v1
      3 resources:
      4   - resources:
      5     - secrets
      6     providers:
      7     - aescbc:
      8         keys:
      9         - name: key1
     10           secret: 3djYtAK9O+7pBbo5oAFKMJAqj4GQS10IhXEXYeMbRKw=      
     11         - name: key2
     12           secret: yXNoEPj4Rv/S/AhctTCLd0ooTN/F1rPk1DPU4gFyY28=
     13     - identity: {}
```



키 적용 : apiserver 재시작

```
# docker stop $(docker ps | grep k8s_apiserver_k8s-master | gawk '{print $1}')
c1652668f5c0
```



키  순서 변경

```
# vi encryption-config.yaml

      1 kind: EncryptionConfig
      2 apiVersion: v1
      3 resources:
      4   - resources:
      5     - secrets
      6     providers:
      7     - aescbc:
      8         keys:
      9         - name: key2
     10           secret: yXNoEPj4Rv/S/AhctTCLd0ooTN/F1rPk1DPU4gFyY28=
     11         - name: key1
     12           secret: 3djYtAK9O+7pBbo5oAFKMJAqj4GQS10IhXEXYeMbRKw=
     13     - identity: {}
```



변경된 키 순서 적용 후, 모든 Secret 암호화 처리

```
# docker stop $(docker ps | grep k8s_apiserver_k8s-master | gawk '{print $1}')
3cb276eead8e

# kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```



결과 

* k8s:enc:aescbc:v1:key2

```
# etcdctl3 get -w fields /registry/secrets/kube-system/platform-auth-idp-credentials

"ClusterID" : 18198080107246212393
"MemberID" : 11619985723144134093
"Revision" : 467731
"RaftTerm" : 2
"Key" : "/registry/secrets/kube-system/platform-auth-idp-credentials"
"CreateRevision" : 3003
"ModRevision" : 467688
"Version" : 3
"Value" : "k8s:enc:aescbc:v1:key2:\xf1\x1c\xb7ڰ\x82U\xa8\x0e\xdd8\xda\xc1\xe8*TTd\xbbO\x8c\x86b\xe3\x90M\v\xe5\xe5#\x82y\x8ch\a%\x1c\xfeKɭ\xc14-\xdd\x15+\"\x02\xa3\xad\xd7\x1du\x82\xb7±\x01\a\\\x13>\xa6\xf9'9\xd22\xf8\x1e\xb3\xd4\xe8m\"\x1dC\xeeQ\".\x1b\xa6[\x8e\x91[\x1aq\xace\x82\x01_\x0fȂ\t߭\xadB\xaa \xe7\xed*I\xd3cA\xe0'\xafi\xc7\xf9\xfc>\xe5\xe3\xbc\xd61Wٝ\vT\x1d\x9fz.\x9e\xe7\u05fa\x83\xf6y\x910\xf98\x1d\x12\xf0\x19\xa0_*b\\\x18\x02\xac\x92\xf8\xc1\x12V\xd6\xc4\xe5\xbb\xd0<I\xe4\to|\xb8\xeb\u007f\xf9\xed.\x9f\x91\xf5@|\x04\x1d.\xa4\xb0\xcey]\x1e1fN\b\xaf\x92b\xd9cMݯlk\a"
"Lease" : 0
"More" : false
"Count" : 1

```



### etcd 내 모든 Secret  복호화



설정 변경 : 7 라인 identity: {} 를 첫번째 providers로 변경

```
# vi encryption-config.yaml

      1 kind: EncryptionConfig
      2 apiVersion: v1
      3 resources:
      4   - resources:
      5     - secrets
      6     providers:
      7     - identity: {}
      8     - aescbc:
      9         keys:
     10         - name: key2
     11           secret: yXNoEPj4Rv/S/AhctTCLd0ooTN/F1rPk1DPU4gFyY28=
     12         - name: key1
     13           secret: 3djYtAK9O+7pBbo5oAFKMJAqj4GQS10IhXEXYeMbRKw=
```



apiserver 재시작

```
# docker stop $(docker ps | grep k8s_apiserver_k8s-master | gawk '{print $1}')
7ac366de2d52
```



모든 secret 복호화

```
# kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```



결과

* k8s

```
# etcdctl3 get -w fields /registry/secrets/kube-system/platform-auth-idp-credentials

"ClusterID" : 18198080107246212393
"MemberID" : 11619985723144134093
"Revision" : 470368
"RaftTerm" : 2
"Key" : "/registry/secrets/kube-system/platform-auth-idp-credentials"
"CreateRevision" : 3003
"ModRevision" : 470320
"Version" : 4
"Value" : "k8s\x00\n\f\n\x02v1\x12\x06Secret\x12\xa2\x01\nf\n\x1dplatform-auth-idp-credentials\x12\x00\x1a\vkube-system\"\x00*$d6eb8612-1564-11e9-b060-06357399dad62\x008\x00B\b\b\xb5\xdd\xe0\xe1\x05\x10\x00z\x00\x12\x17\n\x0eadmin_password\x12\x05admin\x12\x17\n\x0eadmin_username\x12\x05admin\x1a\x06Opaque\x1a\x00\"\x00"
"Lease" : 0
"More" : false
"Count" : 1
```

