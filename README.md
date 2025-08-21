# 🔐 Root Cluster (Teleport Bastion)

이 저장소는 **Teleport 기반 망분리 Kubernetes 환경** 중  **Root Cluster (Bastion)** 구성 코드를 제공합니다.  

Root Cluster는 외부 접속의 **단일 진입점**으로서 Proxy/Auth/Audit 서비스를 제공하며,  Leaf Cluster와 **mTLS Reverse Tunnel**로 연결됩니다.  

---

## 📑 개요
- 중앙 집중형 보안 접속 관리 체계 (Root → Leaf)
- RBAC 이중 통제 (Teleport ↔ Kubernetes)
- 실시간 감사/모니터링 (Logstash + OpenSearch)

---

## 🏗 아키텍처 (Root 관점)

```
[User] → tsh login → Root Proxy ↔ Auth
                         ↓ mTLS Tunnel
                      Leaf Kube Agent → K8s API
```

- **Proxy**: 외부 유일 진입점 (443, 3024–3026)
- **Auth**: 사용자 인증·인가, 세션 관리
- **Audit**: 모든 접속 로그 수집 → Logstash → OpenSearch

---

## ⚙️ 요구사항
- **OS**: Ubuntu 24.04 LTS  
- **Teleport**: v17.5.6 (OSS)  
- **Ports**: 443, 3024, 3025, 3026 (인바운드 허용)  
- **CLI 툴**: `tsh`, `tctl`, `kubectl`  

---

## 🚀 설치 및 실행

### 1. Teleport 설치
```bash
./scripts/install-teleport.sh
```

### 2. 설정 배포
```bash
sudo cp teleport/teleport.yaml /etc/teleport.yaml
sudo systemctl enable --now teleport
```

### 3. 사용자 추가
```bash
tctl users add dev1 --roles=dev
```

---

## 📝 주요 설정 파일
- `teleport/teleport.yaml`  
  Root Cluster용 Proxy/Auth/Audit 설정  

- `role/*.yaml`  
  Teleport Role 정의 (예: app, db, dmz, mgmt)  

---

## 🔑 빠른 시작 (접속 플로우)

### Root Cluster 로그인
```bash
tsh login --proxy=https://<ROOT_DNS_OR_IP>:443
```

### Leaf Cluster 선택
```bash
tsh kube login <leaf-cluster-name>
```

### Kubernetes 접근
```bash
kubectl get ns
```

- ✅ 권한 내 접근 → 정상 동작  
- ❌ 권한 외 접근 → `Forbidden` 에러  

---

## 🛡 RBAC 이중 연동 개요

- **Teleport Role** → `kubernetes_groups` 매핑  
- **K8s RoleBinding**에서 실제 권한 제한  

### Teleport Role (예시: `role/app-role.yaml`)
```yaml
allow:
  kubernetes_groups: ["app-group"]
```

### Kubernetes RoleBinding (예시)
```yaml
subjects:
- kind: Group
  name: app-group
roleRef:
  kind: Role
  name: ns-app-readonly
```

---

## 📂 디렉토리 구조

```
.
├─ .idea/                     # 개발 환경 설정 (IntelliJ 등)
│   ├─ .gitignore
│   ├─ misc.xml
│   ├─ modules.xml
│   ├─ public-cluster.iml
│   └─ vcs.xml
│
├─ role/                      # Teleport Role 정의
│   ├─ app-role.yaml
│   ├─ db-exec-role.yaml
│   ├─ db-role.yaml
│   ├─ dmz-role.yaml
│   └─ mgmt-role.yaml
│
├─ teleport/                  # Teleport Root 설정
│   └─ teleport.yaml
│
└─ docs/
    └─ overview.md            # 전체 프로젝트 개요 (별도 문서)
```
