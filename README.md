# Root Cluster (Teleport Proxy/Auth)

## 1. 개요
Root Cluster는 **망분리 환경의 보안 게이트웨이** 역할을 수행합니다.  
외부 사용자가 내부 Kubernetes(Leaf Cluster)에 접근할 수 있는 **유일한 진입점(Bastion)**이며, 다음 기능을 담당합니다:

- **Teleport Proxy**: 사용자의 K8s/SSH 요청을 수신하고 Leaf Cluster로 안전하게 중계  
- **Teleport Auth**: 사용자 인증·인가 및 RBAC(Role-Based Access Control) 검증  
- **Audit Logging**: 모든 접속·명령·세션 이벤트를 중앙에서 기록 및 전송(Logstash → OpenSearch)  

---

## 2. 네트워크 구조
- 외부에서 노출되는 포트는 최소화 (보안 그룹 설정 기준)
  - `443`: Teleport Proxy (Web/UI, K8s Proxy)  
  - `3024`: Reverse Tunnel (Root ↔ Leaf 연결)  
  - `3025`: Auth Service  
  - `3026`: Kubernetes Proxy  
- `3022`, `3023`: Teleport 내부 노드 간 통신에 사용됨 (Root 내부 전용)  

---

## 3. 설치 절차

### (1) Teleport 설치
```bash
# Ubuntu 24.04 기준
curl https://deb.releases.teleport.dev/teleport-pubkey.gpg | sudo apt-key add -
echo "deb https://deb.releases.teleport.dev/ stable main" | sudo tee /etc/apt/sources.list.d/teleport.list
sudo apt-get update && sudo apt-get install teleport
```

### (2) Teleport 설정 파일 작성
</br>
/etc/teleport.yaml - 실제 운영 값으로 작성
</br>

### (3) 서비스 기동/확인
    sudo systemctl enable teleport
    sudo systemctl start teleport
    sudo systemctl status teleport

    # 상태 및 포트 확인
    sudo tctl status
    sudo ss -tunlp | grep -E ':(443|3024|3025|3026)\b'

## (4) Leaf 등록(요약) — Root에서 토큰 발급, Leaf에서 Helm 설치

### 4.1 Root에서 Kube용 토큰 발급
    sudo tctl tokens add --type=kube --ttl=1h
    # 출력된 토큰값: <YOUR_TOKEN>

### 4.2 Leaf(리프 클러스터)에서 공식 Helm 차트로 Agent 배포
    helm repo add teleport https://charts.releases.teleport.dev
    helm repo update
    # 예시: v17.5.6 사용
    helm install teleport-agent teleport/teleport-kube-agent \
      --version 17.5.6 \
      --namespace teleport --create-namespace \
      --set proxyAddr=rootcluster.store:443 \
      --set authToken=<YOUR_TOKEN> \
      --set kubeClusterName=<YOUR_K8S_CLUSTER> \
      --set roles="kube,app,discovery"


## (5) RBAC 연동 (Teleport ↔ Kubernetes)

### 5.1 Teleport Role (예: app-role) — Root에 생성
```yaml
    kind: role
    version: v7
    metadata:
      name: app-role
    spec:
      allow:
        kubernetes_groups:
          - app-group          # K8s RBAC에서 참조할 그룹명
        kubernetes_labels:
          '*': '*'
        kubernetes_resources:
          - kind: '*'
            namespace: app
            name: '*'
            verbs: ['*']
      deny: {}
```
적용:
    tctl create -f app-role.yaml
    # 사용자/팀에 app-role 부여 (UI 또는 tctl users update)

### 5.2 Kubernetes Role/RoleBinding (Leaf에 적용)
`app` 네임스페이스에 읽기 전용 Role과 Binding 예시:

```yaml
kind: role
version: v7
metadata:
  name: app-role
spec:
  allow:
    kubernetes_labels:
      '*' : '*'
    kubernetes_resources:
      - kind: "*"
        namespace: "app"
        name: "*"
        verbs: ["*"]
    kubernetes_groups:
      - app-group
  deny: {}
```

```yaml
# rolebinding-app.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bind-app-group
  namespace: app
subjects:
- kind: Group
  name: app-group           # ← teleport role에서 내려오는 kubernetes_groups
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: app-reader
  apiGroup: rbac.authorization.k8s.io

```

## 6) 접속/검증

### 6.1 로그인 & K8s 자격증명 수령
    tsh login --proxy=<YOUR_ROOT_HOST:443> --auth=local --user=<USER> <YOUR_ROOT_CLUSTER>
    tsh kube login <YOUR_LEAF_CLUSTER>

### 6.2 권한 테스트 (허용/차단 사례)
    # 허용: app NS 조회 (app-role 부여 사용자)
    kubectl -n app get pods

    # 차단: db NS 조회 시 Forbidden 기대
    kubectl -n db get pods
    # → Error from server (Forbidden) ...

   
