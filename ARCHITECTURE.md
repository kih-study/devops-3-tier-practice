# 시스템 아키텍처

## 전체 구조

```
사용자
  │
  ▼
CloudFront (d1avm8q225gi6s.cloudfront.net)
  ├── /api/* → ALB (백엔드 프록시)
  └── /*     → S3 (프론트엔드 정적 파일, OAC)
                │
                ▼
              ALB (Application Load Balancer)
                │  port 8080
                ▼
         Auto Scaling Group (backend-asg)
          ├── EC2 (pri-svc-a) — Node.js + pm2
          └── EC2 (pri-svc-c) — Node.js + pm2
                │
                ▼
         RDS MySQL 8.0 (pri-db-a/c)
         guestbook-db-kih
```

---

## 네트워크 (VPC)

**CIDR**: `172.16.0.0/23` | **리전**: `ap-northeast-2` (서울) | **AZ**: a, c

### 서브넷 구성

| 이름 | CIDR | 유형 | 용도 |
|------|------|------|------|
| pub-elb-a/c | 172.16.0.0/27, 172.16.0.32/27 | Public | ALB |
| pub-nat-a/c | 172.16.0.64/27, 172.16.0.96/27 | Public | NAT Gateway |
| pub-svc-a/c | 172.16.0.128/27, 172.16.0.160/27 | Public | (예비) |
| pri-elb-a/c | 172.16.0.192/27, 172.16.0.224/27 | Private | 내부 ALB용 |
| pri-svc-a/c | 172.16.1.0/27, 172.16.1.32/27 | Private | EC2 백엔드 |
| pri-db-a/c | 172.16.1.64/27, 172.16.1.96/27 | Private | RDS |
| pri-mgmt-a/c | 172.16.1.128/27, 172.16.1.160/27 | Private | 관리용 |

### 라우팅

| 라우트 테이블 | 대상 | 게이트웨이 |
|-------------|------|-----------|
| rt-public | 0.0.0.0/0 | Internet Gateway |
| rt-private-a | 0.0.0.0/0 | NAT Gateway (nat-a) |
| rt-private-c | 0.0.0.0/0 | NAT Gateway (nat-c) |
| rt-db | — | 인터넷 없음 (격리) |

- **S3 Gateway Endpoint**: private 서브넷에서 S3 직접 접근 (NAT 비용 절감)
- **NAT Gateway**: AZ당 1개 (nat-a, nat-c) — EC2가 아웃바운드 인터넷 접근 가능

---

## 컴포넌트

### CloudFront
- 프론트엔드 정적 파일 (S3) + 백엔드 API (ALB) 통합 배포
- OAC(Origin Access Control)로 S3 직접 접근 차단
- `/api/*` 경로 → ALB 포워딩 (CachingDisabled 정책)
- 정적 파일 → S3 (CachingOptimized 정책)

### S3
- 프론트엔드 정적 파일 호스팅
- CloudFront OAC만 접근 허용 (퍼블릭 차단)
- `guestbook-artifacts-kih1015` — CI/CD 배포 아티팩트 저장

### ALB (Application Load Balancer)
- pub-elb-a/c 서브넷 배치
- HTTP:80 리스너 → 타겟 그룹 `tg-kih` (포트 8080)
- 헬스체크: `GET /api/health` (30초 간격)

### EC2 / Auto Scaling Group
- **ASG 이름**: `backend-asg`
- **Launch Template**: `backend-lt` (v4 이상)
  - AMI: Ubuntu 24.04 (`ami-0e4ab31f1847c850c`)
  - 인스턴스 타입: `t3.micro`
  - IAM 프로파일: `backend-ec2-role`
  - User Data: Node.js 20, pm2, CodeDeploy 에이전트 설치 + `.env` 생성
- **배포**: CodeDeploy Blue/Green (ALB 트래픽 전환 방식)
- **앱 실행**: pm2로 `server.js` 관리 (포트 8080)

### RDS MySQL
- **식별자**: `guestbook-db-kih`
- **엔드포인트**: `guestbook-db-kih.cr68wwi2oocw.ap-northeast-2.rds.amazonaws.com`
- 엔진: MySQL 8.0.42 | 인스턴스: `db.t3.micro` | 스토리지: 20GB gp2
- pri-db-a/c 서브넷 배치, 외부 접근 불가

---

## CI/CD 파이프라인

```
GitHub push (backend-cicd 브랜치)
  │
  ▼
GitHub Actions
  1. OIDC로 AWS 인증 (github-actions-backend-role)
  2. backend/ 디렉터리 zip 패키징
  3. S3 업로드 (guestbook-artifacts-kih1015)
  4. CodeDeploy 배포 트리거
  5. 배포 완료 대기 (최대 30분)
```

### CodeDeploy Blue/Green 배포 흐름

```
배포 시작
  │
  ├─ [1단계] Green ASG 생성 (backend-asg 복제, 새 인스턴스 시작)
  │
  ├─ [2단계] Green 인스턴스에 코드 배포
  │     ApplicationStop → DownloadBundle → BeforeInstall
  │     → Install → AfterInstall → ApplicationStart → ValidateService
  │
  ├─ [3단계] ALB 트래픽을 Green으로 전환
  │     BeforeAllowTraffic → AllowTraffic → AfterAllowTraffic
  │
  ├─ [4단계] Blue 인스턴스 트래픽 차단
  │     BeforeBlockTraffic → BlockTraffic → AfterBlockTraffic
  │
  └─ [5단계] Blue ASG 종료
```

### appspec.yml 훅

| 훅 | 스크립트 | 설명 |
|----|---------|------|
| ApplicationStop | `scripts/application_stop.sh` | 앱 중지 |
| BeforeInstall | `scripts/before_install.sh` | 디렉터리 준비 |
| AfterInstall | `scripts/after_install.sh` | `npm ci` 의존성 설치, `.env` 존재 확인 |
| ApplicationStart | `scripts/application_start.sh` | pm2 start/reload |
| ValidateService | `scripts/validate_service.sh` | 서비스 정상 확인 |

---

## IAM 구성

### github-actions-backend-role
- **신뢰**: `token.actions.githubusercontent.com` (OIDC)
- **권한**:
  - S3: `PutObject / GetObject / GetObjectVersion / ListBucket` → 아티팩트 버킷
  - CodeDeploy: `CreateDeployment / GetDeployment / GetDeploymentConfig / RegisterApplicationRevision / GetApplication / GetApplicationRevision`

### codedeploy-service-role
- **신뢰**: `codedeploy.amazonaws.com`
- **권한**:
  - 관리형: `AWSCodeDeployRole` (ELB, EC2 Describe, 기본 Auto Scaling)
  - 인라인 `BlueGreenExtraPermissions`: `autoscaling:*` + EC2 실행/조회
  - 인라인 `AllowPassBackendEc2Role`: `iam:PassRole` → `backend-ec2-role`

### backend-ec2-role (EC2 인스턴스 프로파일)
- `AmazonEC2RoleforAWSCodeDeploy` — CodeDeploy 에이전트 인증
- `AmazonSSMManagedInstanceCore` — SSM Session Manager 접속
- `AmazonEC2FullAccess` — EC2 조회

---

## 백엔드 애플리케이션 (Node.js)

**런타임**: Node.js 20 + Express + mysql2 + pm2

| 엔드포인트 | 메서드 | 설명 |
|-----------|--------|------|
| `/api/health` | GET | ALB 헬스체크 (서버 ID, IP 반환) |
| `/api/health/db` | GET | DB 연결 상태 확인 |
| `/api/messages` | GET | 방명록 목록 조회 (최근 50건) |
| `/api/messages` | POST | 방명록 작성 (`name`, `content`) |
| `/api/messages/:id` | DELETE | 방명록 삭제 |

**.env (Launch Template User Data로 생성)**
```
PORT=8080
DB_HOST=guestbook-db-kih.cr68wwi2oocw.ap-northeast-2.rds.amazonaws.com
DB_PORT=3306
DB_USER=admin
DB_PASSWORD=devops123!
DB_NAME=guestbook
```

---

## 인프라 구성 방법

| 항목 | 방법 |
|------|------|
| VPC / 서브넷 / NAT / ALB / RDS / S3 / CloudFront | CloudFormation (`3tier-app-cloudformation-no-ec2.yaml`) |
| Launch Template / ASG / CodeDeploy 앱·배포 그룹 | AWS 콘솔 수동 구성 |
| IAM Role (3개) | AWS 콘솔 수동 구성 |
| CI/CD 워크플로우 | GitHub Actions (`backend-deploy.yml`) |
