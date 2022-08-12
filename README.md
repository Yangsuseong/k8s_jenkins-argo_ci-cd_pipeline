# K8s Jenkins&Argo CI/CD Pipeline 구축 
[Kubernetes 위에 Jenkins 설치하기](https://cwal.tistory.com/20) 참고
argoCD - [ArgoCD: Kubernetes에 GitOps 적용하기](https://cwal.tistory.com/19) 참고
kustomizer - https://cwal.tistory.com/22 참고

[image:31720BDD-885C-4482-B51B-C98EC8717142-14520-0000054BC98B2A98/C489D3EB-DD2D-41AD-BB1E-D0B585975F4B.png]




## K8s에 Jenkins 배포
---

::jenkins 서비스가 배포될 노드에 /jenkins-data 디렉토리 생성::
```bash
$ mkdir /jenkins-data
$ chmod 777 /jenkins-data 
```


:::ci namespace 생성:::
```bash
$ kubectl create namespace ci
```


### yaml 생성 후 배포
::jenkins-master.yaml::
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: jenkins
spec:
  serviceName: jenkins
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        ports:
          - name: http-port
            containerPort: 8080
          - name: jnlp-port
            containerPort: 50000
        volumeMounts:
          - name: jenkins-vol
            mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-vol
          hostPath:
            path: /jenkins-data
            type: Directory
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: jenkins
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-jnlp
spec:
  type: ClusterIP
  ports:
    - port: 50000
      targetPort: 50000
  selector:
    app: jenkins
```

::jenkins-rbac.yaml::
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: jenkins
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get","list","watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jenkins
subjects:
- kind: ServiceAccount
  name: jenkins
```

::배포::
```bash
$ kubectl apply -n ci -f jenkins-rbac.yaml
$ kubectl apply -n ci -f jenkins-master.yaml
```




## Jenkins Dashboard 
---

::NodePort 확인::
```bash
$ kubectl get svc -n ci
>>>
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
jenkins        NodePort    10.105.178.161   <none>        8080:30146/TCP   67m
jenkins-jnlp   ClusterIP   10.111.24.33     <none>        50000/TCP        67m
```

웹 브라우저에서 {Master IP}:30146 으로 접속

::Dashboard Token 확인::
```bash
$ kubectl exec -it -n ci jenkins-0 -- cat /var/jenkins_home/secrets/initialAdminPassword
>>> 43b5f6b6be7d4bb3b325e8de6e79d17e
```

::Install suggested plugin 클릭::
자동으로 필요한 패키지가 설치됨 (5~10	분정도 소요)
[image:0BE95933-D2C6-4243-A87F-A8D3AEAE0FD2-14520-0000056DB90C7377/스크린샷 2022-08-05 오후 1.11.17.png]

::설치 이후 나오는 화면에서 admin 계정 설정::
테스트 환경에서는 아래와 같이 생성함
	* ID: admin
	* PW: miruware!


::Jenkins 관리 -> 플러그인관리 -> 설치가능 에서 kubernetes와 Docker Pipeline 플러그인 검색하여 설치::
[image:F1ED0ADD-D2CC-44F9-AD70-27604E78F1DB-14520-0000056DB90D365C/스크린샷 2022-08-05 오후 1.24.06.png]


::Jenkins 관리 -> 노드 설정 -> Configure Clouds 에서 Kubernetes 선택::
[image:67BDFB69-D642-4D54-A0AA-A07AF5062D96-14520-0000056DB90DA879/2F163855-C6FB-4800-959B-5963DE3C1A70.png]

	* Name: 해당 클러스터를 구분할 수 있는 이름
		* kubernetes
	* Kubernetes URL: Jenkins가 k8s 내부에서 실행중이므로, API서버의 In-cluster URL을 입력
		* https://kubernetes.default.svc
	* Kubernetes Namespace: Remote Agent 관련 리소스가 생성될 Namespace를 의미한다. RBAC 설정시 ‘ci’ Namespace에만 유효하도록 정의하였으므로 ci를 입력
		* ci
	* Jenkins URL: In-cluster의 Jenkins URL로 HTTP 관련 Service 이름과 Port를 명시
		* http://jenkins:8080
	* Jenkins tunnel: Remote Agent가 접근할 주소, jnlp 관련  Service 이름과 Port를 입력
		* jenkins-jnlp:50000
	* Pod Labels 추가
		* Key: jenkins
		* Value: slave 

 
::나머지 항목은 입력하지 않고 비워두거나 기본값으로 설정.::
Test Connection 버튼을 눌러서 k8s 연결에 문제가 없는지 확인 후 Save 버튼을 눌러서 위 설정을 적용
[image:69106094-A0FC-4BDF-BB73-26FEB1DCE8B9-14520-0000056DB90E02E6/스크린샷 2022-08-05 오후 1.39.05.png]



## Jenkins와 Github 연동
---

::Jenkins Master 서버의 Container 안으로 들어가 SSH Key 생성::
```bash
$ kubectl exec -ti -n ci jenkins-0 -- bash

(jenkins-0)$ ssh-keygen
```


### <Git Hub>
::SSH key(id_rsa.pub) 확인::
```bash
(jenkins-0)$ cat /var/jenkins_home/.ssh/id_rsa.pub
>>>
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDHGL2W6nflo43IBJXCdoIhpeBtb/zPppBjZYFnlpxAVgsPKdK146KuYpATo4RbXFSC+uE9X3J7uVP6aG7rlQNErjBgXOqLYgWdxyXh3gdSyfOL4IVSl22Xrc3HINDjX+xee8J43i9AQe9+kjD3imQ8CqcdwW03owmbEUs/wxt9xgbyQPwNGG/Yv/SPZwgVjvTeie9YUDOBAFJthkv03xEdKeGAU2bdUg6XpKVoY7tGgx8pHS2fcKPflkPK25q9gBwQvPkdSI7RQ+/0/nJ+xEQ1OY0U4r6pl2gEvU8Ycob3QxKcW3OT//8HbYdPsk9gxbVwJDfNN243WdVfHSduVeTeKP+Pi++eX5/KC2FxNAKKar9IOKr5lHNjWuaNL0PaY7VzbPehWA1NTNjZ105Vc+fP47YjO8SKfQumAmXLB4DOJOfeXZcTXjA9oXNuO+7Af/0ZKypqeGpIjpFNGRCSG3F6cFfyBBKbBrNd7yV36lLmNKtMtDF3t0vDG3JxyT8IZ00= jenkins@jenkins-0
```

::Git Hub 홈페이지 Setting -> SSH and GPG keys 탭에서 SSH key 등록::
[image:8393E62B-283E-4B6C-8875-15DD4A9466CF-14520-0000056DB90ED63C/스크린샷 2022-08-05 오후 1.49.42.png]


### <Jenkins>
::SSH Key(id_rsa) 확인::
```bash
(jenkins-0)$ cat /var/jenkins_home/.ssh/id_rsa
>>>
-----BEGIN OPENSSH PRIVATE KEY-----
#$%#$%#$%#$%#$%!@#
#$!@#^#$%^@#$%!@#$
...
!@#$#$#$%!@%#$%!@#
!@#$==
-----END OPENSSH PRIVATE KEY-----
```


::Jenkins Dashboard 관리 -> Manage Credentials 로 이동 후 Credentials 등록::
[image:B4C44A7D-CBFA-4DA9-ABD2-DD0F194FF22E-14520-0000056DB90F4F7B/스크린샷 2022-08-05 오후 2.24.58.png]

[image:89D7AA9C-A15B-4793-90C6-CFE9E6EF01A5-14520-0000056DB90FC4D4/스크린샷 2022-08-05 오후 4.33.18.png]
	* Credential ID 와 Private Key 값만 입력하고 나머지는 공백
		* ID : Git hub ID
		* Key: 위에서 확인한 id_rsa 값 붙여넣기

::Jenkins 관리 -> Configure Global Security 로 이동::
Jenkins 2.346.2 버전 기준 최 하단의
Git Host Key Verification Configuration 을
`Accept first connection`
으로 변경 



### <Docker Hub>
::Jenkins Dashboard 관리 -> Manage Credentials 로 이동 후 Credentials 등록::
[image:CA3A9816-1B49-4388-A7C2-23FBF4E12BE0-14520-0000056DB910AA43/스크린샷 2022-08-05 오후 2.24.58.png]
[image:57F7B71D-71A5-4053-B7B6-1154592F1F76-14520-0000056DB9112F06/스크린샷 2022-08-08 오후 12.15.14.png]


	* Docker hub Username/Password  입력
	::ID : Jenkins에서 변수로 사용할 ID::

	나머지는 빈칸으로 둔다.



## Git Repository 추가
---
[GitHub - learnk8s/docker-hello-world: Node.js “Hello world”](https://github.com/learnk8s/docker-hello-world)
위 링크에 있는 코드를 Fork 하여 테스트를 진행한다.


::Git clone 후 Jenkinsfile 을 생성하여 Commit&Push::
```bash
$ git clone https://github.com/Yangsuseong/docker-hello-world.git

$ vim Jenkinsfile
```

::Jenkinsfile::
```vim
podTemplate(label: 'docker-build', 
  containers: [
    containerTemplate(
      name: 'git',
      image: 'alpine/git',
      command: 'cat',
      ttyEnabled: true
    ),
    containerTemplate(
      name: 'docker',
      image: 'docker',
      command: 'cat',
      ttyEnabled: true
    ),
  ],
  volumes: [ 
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'), 
  ]
) {
    node('docker-build') {
        def dockerHubCred = <your_dockerhub_cred>
        def appImage
        
        stage('Checkout'){
            container('git'){
                checkout scm
            }
        }
        
        stage('Build'){
            container('docker'){
                script {
                    appImage = docker.build("<your-dockerhub-id>/node-hello-world")
                }
            }
        }
        
        stage('Test'){
            container('docker'){
                script {
                    appImage.inside {
                        sh 'npm install'
                        sh 'npm test'
                    }
                }
            }
        }

        stage('Push'){
            container('docker'){
                script {
                    docker.withRegistry('https://registry.hub.docker.com', '<Your Dockerhub Cred>'){
                        appImage.push("${env.BUILD_NUMBER}")
                        appImage.push("latest")
                    }
                }
            }
        }
    }
    
}
```

* 변경해야하는 변수
	* <your-dockerhub-id>
	* <Your Dockerhub Cred>

::commit & push::
```bash
$ git add Jenkinsfile
$ git commit -a
$ git push
```




## Jenkins Pipeline Job 추가
---

### Jenkins Dashboard -> 새로운 Item 에서 Job 추가

::Job 이름 및 종류 선택::
[image:987AAFDA-844F-4962-A611-CF44B4C32CA7-14520-0000056DB911A0A2/스크린샷 2022-08-05 오후 3.09.48.png]

::Job 구성::
[image:BCF7ACD0-8FD2-45B2-974D-CA52731D5FA9-14520-0000056DB9120C5B/스크린샷 2022-08-08 오후 1.40.30.png]
	* GitHub project: 빌드할 소스코드와 Jenkinsfile이 위치한 GitHub 프로젝트 URL(.git 생략)
	* Github hook trigger for GITScm polling 체크
		* git push 가 이루어질때마다 자동으로 빌드 진행하도록 하는 trigger

(notice) 
Poll SCM을 체크하고 ‘H/5 * * * *’ 을 입력 시 5분마다 빌드가 진행되도록 설정할 수 있음.


::Pipeline 설정::
[image:D32C0F3E-C286-4559-9081-7A4E89944EE9-14520-0000056DB912767C/스크린샷 2022-08-05 오후 3.18.47.png]
[image:7696712C-A6D2-48F4-8040-DD96ADC6436D-14520-0000056DB912FB95/스크린샷 2022-08-05 오후 3.19.07.png]


	* 체크아웃할 Repository에 이미 Pipeline Script이 존재하므로 ‘Pipeline script from SCM’ 항목을 선택
	* Repository URL은 해당 Repository SSH URL(.git 포함)을 입력
	* Credential은 미리 생성한 GitHub Credential을 사용
	* 빌드할 대상 브랜치는 master


### Github -> Repository -> Setting -> Webhooks 설정

::[Add Webhook] 클릭하여 web hook 생성::
[image:F6033D73-8150-45B0-882E-DB36122BE62F-14520-0000056DB9135EEE/스크린샷 2022-08-08 오후 1.43.29.png]
	* Payload URL
		* http://{Jenkins 를 외부에서 접속할 수 있는 Domain}/github-webhook/
	* Content type
		* application/json



### Web hook 정상작동 확인

::Jenkins와 연동되어 있는 Repository에 코드를 수정하여 Push::
[image:8B99C5E6-0461-444F-A525-F177062DCBB0-14520-0000056DB913BA71/스크린샷 2022-08-08 오후 1.46.33.png]
 
::Github 홈페이지 -> Repository -> Setting -> Webhooks -> 생성한 web hook 에서 Recent Deliveries 클릭하여 확인::
[image:9C286238-45B2-40F8-841B-BB3DC2F13B14-14520-0000056DB9141DA9/스크린샷 2022-08-08 오후 1.47.51.png]

::Jenkins 작업에서 Build가 이루어졌는지 확인::
[image:E85D44B6-8AEA-48D3-83F2-756F9E3CF4F1-14520-0000056DB914778C/스크린샷 2022-08-08 오후 1.48.52.png]




## Jenkins Dashboard 에서 ssh agent 설치
---
::Jenkins 관리 -> 플러그인 관리 -> 설치가능 -> ssh agent::
[image:109CB64A-268A-4885-8AFB-C102AAC8B927-14520-0000054C09C01557/스크린샷 2022-08-09 오후 2.28.15.png]

ssh Agent 설치 진행




## ArgoCD 배포
---

### Argo CD 설치

::Argo CD 배포용 namespace 생성::
```bash
$ kubectl create namespace argocd
```

::Argo CD Manifest 파일 다운로드::
```bash
$ curl https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -o argo-cd.yaml
```

::Manifest 파일 수정::
현재 테스트 환경의 k8s는 Desktop의 VM 위에 구축되었기 때문에, Load Balancer 대신 NodePort를 사용한다.
argocd-server 서비스 부분을 찾아 type: NodePort 를 추가한다.
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: argocd-server
    app.kubernetes.io/part-of: argocd
  name: argocd-server
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  - name: https
    port: 443
    protocol: TCP
    targetPort: 8080
  selector:
    app.kubernetes.io/name: argocd-server
```

::Argo CD CLI Tool 다운로드 후 환경변수 추가::
```bash
$ VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')

$ curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64

$ chmod +x /usr/local/bin/argocd
```

::Argo CD 배포::
```bash
$ kubectl apply -n argocd -f argo-cd.yaml 
```

::서비스 포트 확인::
```bash
$ kubectl get svc -n argocd argocd-server
>>>
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
argocd-server   NodePort   10.101.149.21   <none>        80:30817/TCP,443:30499/TCP   5s
```




## Argo CD Dashboard
---
::admin 계정 Default  Password 확인::
계정 : admin
```bash
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
>>> 5ENdc9r24v83djXH
```

::admin 계정의 비밀번호를 변경::
```bash
$ argocd login <ARGOCD_SERVER> 
#ex) argocd login localhost:30817

$ arcocd account update-password
>>> 
*** Enter password of currently logged in user (admin):
*** Enter new password for user admin:
*** Confirm new password for user admin:
Password updated
Context 'localhost:30953' updated
```




## Kustomizer 설치 및 사용 준비
---
* 테스트용 Hello world 디렉토리와 kustomizer 디렉토리는 다른 repository 에서 관리되어야 한다.
	* 만약 같은 디렉토리에 있으면?
		* Tag 변경을 위해 지속적으로 Git push 가 이루어지므로 이미지 빌드가 무한 루프가 돈다.
* 이번 테스트에서는 같은 Git Repository에 관리를 진행하여 무한 루프가 도는 것을 확인할 수 있다.

::kustomizer 기본 디렉토리 구조::
[image:323E5EC7-2B4D-4D34-8DDB-13E351951C3E-14520-0000054C09C0909B/스크린샷 2022-08-09 오후 2.07.27.png]


### Manifest 작성
::./base/deployment.yaml::
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deployment
  labels:
    app: hello
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: <Docker Hub 계정>/node-hello-world:latest
        ports:
        - containerPort: 8080
```

::./base/service.yaml::
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  type: NodePort
  selector:
    app: hello
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

::./base/kustomization.yaml::
```yaml
resources:
- deployment.yaml
- service.yaml
```

::./env/dev/deployment.yaml::
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deployment
  labels:
    env: dev
spec:
  replicas: 4
```

::./env/dev/service.yaml::
```yaml
apiVersion: v1
kind: Service
```

::./env/dev/kustomization.yaml::
```yaml
patchesStrategicMerge:
- deployment-patch.yaml
- service-patch.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
images:
- name: <Docker Hub 계정>/node-hello-world
  newTag: "278"
```

prod 디렉토리는 이번 테스트에서는 없어도 됨.


### Jenkinsfile 수정
./env/dev/kustomization.yaml 파일의 newTag 값이 자동으로 변경될 수 있도록
Git Repository 내의 Jenkinsfile을 수정한다.

::Jenkinsfile::
```vim
podTemplate(label: 'docker-build',
  containers: [
    containerTemplate(
      name: 'docker',
      image: 'docker',
      command: 'cat',
      ttyEnabled: true
    ),
    containerTemplate(
      name: 'argo',
      image: 'argoproj/argo-cd-ci-builder:latest',
      command: 'cat',
      ttyEnabled: true
    ),
  ],
  volumes: [
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
  ]
) {
    node('docker-build') {
        def dockerHubCred = "dockerhub_cred"
        def appImage

        stage('Checkout'){
            container('argo'){
                checkout scm
            }
        }

        stage('Build'){
            container('docker'){
                script {
                    appImage = docker.build("<Docker Hub 계정>/node-hello-world")
                }
            }
        }

        stage('Test'){
            container('docker'){
                script {
                    appImage.inside {
                        sh 'npm install'
                        sh 'npm test'
                    }
                }
            }
        }

        stage('Push'){
            container('docker'){
                script {
                    docker.withRegistry('https://registry.hub.docker.com', '<Jenkins Docker 인증서 이름>'){
                        appImage.push("${env.BUILD_NUMBER}")
                        appImage.push("latest")
                    }
                }
            }
        }

        stage('Deploy'){
            container('argo'){
                checkout([$class: 'GitSCM',
                        branches: [[name: '*/master' ]],
                        extensions: scm.extensions,
                        userRemoteConfigs: [[
                            url: '<Git Hub 주소>',
                            credentialsId: '<Jenkins Git Hub 인증서 이름>',
                        ]]
                ])
                sshagent(credentials: ['<Git Hub ssh 인증서 이름>']){
                    sh("""
                        #!/usr/bin/env bash
                        set +x
                        export GIT_SSH_COMMAND="ssh -oStrictHostKeyChecking=no"
                        git config --global user.email "<Git Hub ID>"
                        git checkout master
                        cd env/dev && kustomize edit set image tntjd5596/node-hello-world-test:${BUILD_NUMBER}
                        git commit -a -m "updated the image tag"
                        git push
                    """)
                }
            }
        }
    }
}
```





## Jenkins 이미지 빌드 정상 작동 확인
---
[image:ECF22708-C569-4FB6-9CA9-818F975B27AD-14520-0000054C09C0BFEC/스크린샷 2022-08-09 오후 2.29.35.png]

::빌드 완료 후 Git Hub Repository의 ./env/dev/kustomization.yaml 확인::
[image:D87A706F-A100-4205-AD9E-23611CEB0D37-14520-0000054C09C0E795/스크린샷 2022-08-09 오후 2.31.21.png]

newTag 가 최신 빌드 버전(304)으로 변경된 것을 확인할 수 있다.




## Argo Application 배포
---

::Argo Dashboard 에서 [+ NEW APP] 버튼 클릭::

[GENERAL]
[image:C3E05EE4-C38A-469E-9DA7-50F80FABC980-14520-0000054C09C150C6/스크린샷 2022-08-09 오후 2.37.16.png]

* Application Name
	* <생성할 Application 이름>
* Project Name
	* default
* SYNC POLICY
	* 자동으로 지속적인 배포가 필요하므로 Automatic 선택
+ SELF HEAL
+ AUTO-CREATE NAMESPACE
체크

[SOURCE]
[image:541ABAC4-E7FD-4285-B896-2EFD5CF4F829-14520-0000054C09C17434/스크린샷 2022-08-09 오후 2.39.43.png]

* Repository URL
	* Source 가 있는 Git Hub Url
* Revision
	* HEAD
* Path
	* ./dev/dev 디렉토리의 kustomization.yaml 을 사용할 예정이므로 dev/dev 선택

[DESTINATION]
[image:D4EEFDF1-4140-42E0-8B3E-6C19144870E7-14520-0000054C09C19753/스크린샷 2022-08-09 오후 2.41.55.png]

* Cluster URL
	* https://kubernetes.default.svc
* Namespace
	* APP을 실행할 namespace 이름 (이번 테스트에서는 dev 사용)

[Kustomize] 를 클릭하여 [Diretory] 로 변경
[image:13051D0F-0755-421F-8045-FCBA6BB5B839-14520-0000054C09C1BA9B/스크린샷 2022-08-09 오후 2.43.55.png]




## Argo App 상태 확인
---
::Argo Dashboard Main::
[image:7C24FB06-2AC9-41F1-83CB-16A944B202CD-14520-0000054C09C1DCFC/스크린샷 2022-08-09 오후 2.47.26.png]

::Hello-world-app 클릭하여 상세정보로 이동::
[image:757C6D99-C937-4483-95DE-608AF87E7EDD-14520-0000054C09C1FF38/스크린샷 2022-08-09 오전 11.19.41.png]

::Kubernetes 터미널에서 배포중인 Hello Service에 Curl 요청::
```bash
$ kubectl get svc -n dev
>>>
NAME    TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
hello   NodePort   10.102.234.143   <none>        80:30577/TCP   68m

$ curl localhost:30577
>>> Hello World!!
```




## Jenkins와 Argo 의 CI/CD Pipeline 작동 확인
---

::Git Repository의 index.js 파일 내용 변경::
```vim
const express = require('express')
const app = express()
const port = process.env.PORT || 8080;

app.get('/', (req, res) => {
  res.send('Hello World!! This Is Trigger Test') # 내용 추가
})

app.get('/version', (req, res) => {
  res.send(process.env.VERSION || 'No version')
})

app.listen(port, () => {
  console.log(`Example app listening on port ${port}!`)
})
```

::Git Push::
```bash
$ git pull
$ git add index.js
$ git commit
$ git push
```

::Jenkins 대시보드에서 이미지 Build 확인::
[image:CF40D6F4-82D7-4C6C-82C3-E2EBD161AAAF-14520-0000054C09C23C55/스크린샷 2022-08-09 오후 3.00.35.png]

::빌드 완료 후 Argo Dashboard 에서 Application 상태 확인::
[image:C5C204AA-F442-488D-9E41-6D7DDC9E00FE-14520-0000054C09C25FE4/스크린샷 2022-08-09 오후 2.53.18.png]

* 자동으로 변경된 내용으로 배포가 진행되는 것을 확인

::Kubernetes 터미널에서 배포중인 Hello Service에 Curl 요청::
```bash
$ kubectl get svc -n dev
>>>
NAME    TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
hello   NodePort   10.102.234.143   <none>        80:30577/TCP   68m

$ curl localhost:30577
>>> Hello World!! This Is Trigger Test
```
