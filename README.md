# 🔐 Root Cluster (Teleport Bastion)

이 저장소는 **망분리 Kubernetes 환경**에서 **외부 접속의 단일 진입점** 역할을 수행하는  **Teleport Root Cluster (Bastion)** 구성 코드를 제공합니다.  

Root Cluster는 **Proxy/Auth/Audit 서비스**를 포함하며,  Leaf Cluster와 **mTLS Reverse Tunnel**로 안전하게 연결됩니다.  

---

## 📑 개요
- 중앙 집중형 보안 접속 관리 체계의 핵심 진입점  
- RBAC 이중 통제( Teleport ↔ Kubernetes )를 위한 Role 정의  
- 모든 사용자 세션/명령어 실행 로그를 수집하여 **Logstash + OpenSearch**로 연계  
- API Server 직접 노출 차단, Zero-Trust 기반 접근 제어  

---

## 🌐 전체 아키텍처에서의 Root Cluster 역할

Root Cluster는 단순한 게이트웨이가 아니라, **망분리 환경 전체의 보안·관제 허브**입니다.  
모든 사용자의 접근은 반드시 Root를 통과해야 하며, Root는 다음과 같은 기능을 전담합니다:

1. **Bastion 역할**  
   - 외부에서 접속 가능한 **유일한 진입점**  
   - API Server는 직접 노출되지 않음 (Leaf는 사설망에 존재)  

2. **중앙 인증·인가**  
   - 모든 사용자 세션은 Root Auth 서비스에서 검증  
   - Teleport Role ↔ Kubernetes RBAC을 매핑하여 **이중 권한 검증** 수행  

3. **감사 및 로깅 허브**  
   - 모든 세션/명령 로그를 Audit 서비스가 수집  
   - Logstash → OpenSearch로 전달되어 **중앙 집중형 모니터링** 구축  

4. **보안 통제 지점**  
   - 불필요한 인바운드 포트 차단 (443, 3024~3026만 개방)  
   - mTLS 기반 터널로 Leaf와 안전하게 연결  
   - 내부자 위협·권한 남용 차단을 위한 RBAC 및 로그 기반 탐지 수행  

> **즉, Root Cluster = “모든 접속이 시작되고, 모든 보안 통제가 이루어지며,  
> 모든 감사 로그가 집계되는 보안 핵심 지점”**입니다.

---

## 🏗 아키텍처 (Root 관점)

```
[User] → tsh login → Root Proxy ↔ Auth
                         ↓ mTLS Tunnel
                      Leaf Kube Agent → K8s API
```

- **Proxy**: 외부 사용자가 접속하는 유일 진입점 (443, 3024–3026)  
- **Auth**: 사용자 인증·인가, RBAC 정책 적용, 세션 관리  
- **Audit**: 모든 접속/명령 로그 기록 → Logstash → OpenSearch  

> Root Cluster는 외부망에 위치하며 Bastion 역할을 수행합니다.  
> Leaf Cluster(K3s)는 사설망에 배치되어 Root와의 mTLS 연결로만 접근 가능합니다.

---

## ⚙️ 요구사항
- **OS**: Ubuntu 24.04 LTS (AWS EC2 t3.large 이상 권장)  
- **Teleport**: v17.5.6 (Open Source Edition)  
- **Ports**: 443, 3024, 3025, 3026 (보안 그룹 허용)  
- **CLI 툴**: `tsh`, `tctl`, `kubectl`  
- **네트워크**: 방화벽으로 불필요한 인바운드 차단 → Root만 외부 노출  

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
  - Proxy/Auth/Audit 서비스 설정 포함  
- `role/*.yaml`  
  - Root Cluster Teleport Role 정의 (예: app, db, dmz, mgmt)  

## 🛡 RBAC 이중 연동
- Teleport Role → `kubernetes_groups` 매핑  
- Kubernetes RoleBinding → 네임스페이스 단위 최소 권한 제어  

예시:
```yaml
# Teleport Role (role/app-role.yaml)
allow:
  kubernetes_groups: ["app-group"]
```

```yaml
# Kubernetes RoleBinding
subjects:
- kind: Group
  name: app-group
roleRef:
  kind: Role
  name: ns-app-readonly
```

---

## 📊 감사 로그 & 모니터링
- **Root Audit Logs** → Logstash → OpenSearch → Dashboards  
- 사용자/리소스별 접근 이력 실시간 시각화  
- 비인가 접근, 권한 남용, 비정상 로그인 탐지  
- 감사 로그를 로컬 + 중앙 저장소(OpenSearch)에 이중 보관하여 무결성 보장

---

## 🌟 Root Cluster의 기대 효과
- **보안성 강화**: Root Proxy 단일 노출, API Server 비공개, Pod 단위 RBAC  
- **운영 효율성**: 중앙 집중 인증·인가, 단일 Bastion 관리  
- **규제 대응**: 금융위/금보원 가이드라인 충족(행위 기록·추적, 중앙 감사)  
- **확장성**: 멀티 클러스터/하이브리드 환경 지원  

---

## 📂 디렉토리 구조

```
.
├─ .idea/                     # 개발 환경 설정
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
├─ teleport/                  # Root Cluster 설정
│   └─ teleport.yaml
│
└─ README.md
```
