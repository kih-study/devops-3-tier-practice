# CI/CD 파이프라인 트러블슈팅 기록

GitHub Actions → S3 → CodeDeploy Blue/Green 파이프라인을 구축하면서 발생한 문제와 해결 과정입니다.

---

## 1. S3 업로드 AccessDenied

**오류**
```
An error occurred (AccessDenied) when calling the PutObject operation:
User: .../assumed-role/github-actions-backend-role/GitHubActions is not authorized to perform: s3:PutObject
```

**원인**  
`github-actions-backend-role`에 정책이 전혀 붙어있지 않았음. Role만 생성되고 권한 부여가 누락.

**해결**  
인라인 정책 `GithubActionsBackendDeployPolicy`를 추가.
```json
{
  "s3:PutObject / s3:GetObject": "arn:aws:s3:::guestbook-artifacts-kih1015/*",
  "codedeploy:CreateDeployment / GetDeployment / ...": "guestbook-backend 관련 리소스"
}
```

---

## 2. CodeDeploy 리소스 ARN 불일치

**오류**
```
AccessDeniedException: not authorized to perform: codedeploy:CreateDeployment on resource:
arn:aws:codedeploy:.../deploymentgroup:guestbook-backend/guestbook-backend-dg
```

**원인**  
정책의 CodeDeploy 리소스 ARN이 잘못된 애플리케이션 이름을 가리키고 있었음.

| 항목 | 정책에 등록된 값 | 실제 사용 값 |
|------|----------------|-------------|
| Application | `devops-3tier-backend` | `guestbook-backend` |
| Deployment Group | `devops-3tier-backend-bg` | `guestbook-backend-dg` |

**해결**  
정책의 Resource ARN을 실제 애플리케이션/배포 그룹 이름으로 수정.

---

## 3. IAM_ROLE_PERMISSIONS: AmazonAutoScaling

**오류**
```
code: IAM_ROLE_PERMISSIONS
message: The IAM role .../codedeploy-service-role does not give you permission
         to perform operations in the following AWS service: AmazonAutoScaling.
```

**원인**  
`AWSCodeDeployRole` 관리형 정책이 Blue/Green + ASG 배포에 필요한 일부 Auto Scaling 권한을 누락하고 있음. CodeDeploy가 배포 전 사전 IAM 체크에서 차단.

누락된 주요 액션:
- `autoscaling:SetDesiredCapacity` — Green ASG 스케일업
- `autoscaling:TerminateInstanceInAutoScalingGroup` — Blue 인스턴스 정리
- `autoscaling:DetachLoadBalancerTargetGroups` — Blue ASG를 타겟 그룹에서 분리
- 기타 다수

**해결**  
`codedeploy-service-role`에 인라인 정책 `BlueGreenExtraPermissions` 추가.
```json
{ "Action": "autoscaling:*", "Resource": "*" }
```

> `iam:SimulatePrincipalPolicy`로 검증했을 때 허용으로 나와도 실제 배포가 막혔던 이유:  
> CodeDeploy가 내부적으로 체크하는 액션 중 시뮬레이션에서 확인하지 않은 항목이 있었음.

---

## 4. ec2:RunInstances AccessDenied

**오류** (CloudTrail)
```
RunInstances — Client.UnauthorizedOperation
User: .../assumed-role/codedeploy-service-role/... is not authorized to perform: ec2:RunInstances
```

**원인**  
CodeDeploy Blue/Green은 Green 환경 생성 시 새 EC2 인스턴스를 직접 실행함. `AWSCodeDeployRole`에 `ec2:RunInstances`가 없었음.

**해결**  
`BlueGreenExtraPermissions`에 EC2 권한 추가.
```json
{
  "ec2:RunInstances",
  "ec2:DescribeLaunchTemplates / DescribeLaunchTemplateVersions / GetLaunchTemplateData",
  "ec2:DescribeImages / DescribeInstanceTypes / DescribeSecurityGroups / DescribeSubnets",
  "ec2:CreateTags"
}
```

---

## 5. HEALTH_CONSTRAINTS — CodeDeploy 에이전트 미응답

**오류**
```
CodeDeploy agent was not able to receive the lifecycle event.
Check the CodeDeploy agent logs on your host and make sure the agent is running.
```

**원인 (복합)**

| 항목 | 문제 |
|------|------|
| AMI | Ubuntu 24.04 기본 이미지 — CodeDeploy 에이전트 미포함 |
| Launch Template v1 | User Data 없음 (3바이트 깨진 데이터) |
| Launch Template v2 | User Data 추가됐으나 IAM 인스턴스 프로파일 누락 |
| Launch Template v3 | 프로파일 추가됐으나 `backend-asg`가 `$Default`가 아닌 v2로 고정 |
| `backend-asg` 설정 | 특정 버전으로 고정되어 있어 CodeDeploy가 Green ASG 복제 시 v2 그대로 사용 |

**해결 순서**
1. Launch Template v4 생성 — User Data(에이전트 설치) + `backend-ec2-role` 인스턴스 프로파일 포함
2. `backend-asg` Launch Template 버전을 `$Default`로 변경
3. `backend-asg` Instance Refresh 실행 — 기존 Blue 인스턴스를 v4로 교체

**Ubuntu 24.04 CodeDeploy 에이전트 설치 User Data**
```bash
#!/bin/bash
apt-get update -y
apt-get install -y ruby-full wget
cd /tmp
wget https://aws-codedeploy-ap-northeast-2.s3.ap-northeast-2.amazonaws.com/latest/install
chmod +x ./install
./install auto
systemctl enable codedeploy-agent
systemctl start codedeploy-agent
```

---

## 6. Blue 인스턴스 BeforeBlockTraffic 실패

**오류**
```
BeforeBlockTraffic — Failed
CodeDeploy agent was not able to receive the lifecycle event.
```

**원인**  
기존 Blue 인스턴스(`backend-asg`)가 v1/v2 Launch Template으로 생성되어 CodeDeploy 에이전트가 없었음. `appspec.yml`에 `BeforeBlockTraffic` 훅이 없어도 CodeDeploy는 Blue 인스턴스의 에이전트에 연결을 시도함.

**해결**  
`aws autoscaling start-instance-refresh`로 Blue 인스턴스를 v4로 롤링 교체.

---

## 7. 배포 중 Instance Refresh 동시 실행 (타이밍 충돌)

**현상**  
Blue 인스턴스 교체(Instance Refresh) 중 배포를 실행하자, CodeDeploy가 이미 종료된 인스턴스 ID를 Blue 대상으로 잡아서 실패.

**원인**  
CodeDeploy는 배포 시작 시점에 Blue 인스턴스 목록을 고정함. Instance Refresh가 해당 인스턴스를 종료하면 배포 중 연결 불가.

**해결**  
Instance Refresh 완료를 확인한 후 배포 실행.

---

## 8. 실패한 배포마다 Green ASG 잔존

**현상**  
배포가 실패할 때마다 `CodeDeploy_guestbook-backend-dg_d-XXXXXX` 이름의 ASG가 종료되지 않고 남아 인스턴스가 누적됨 (최대 6개 동시 실행).

**원인**  
CodeDeploy Blue/Green은 배포 성공 시에만 Blue ASG를 종료함. 실패 시 Green ASG는 자동 삭제되지 않음.

**해결 (수동)**
```bash
aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name "CodeDeploy_guestbook-backend-dg_d-XXXXXX" \
  --force-delete
```

---

## 9. 배포 성공 후 500 오류

**현상**  
배포는 성공했으나 서비스 접속 시 HTTP 500.

**원인**  
Launch Template User Data의 `.env` 파일에 RDS 엔드포인트 플레이스홀더가 그대로 남아있었음.
```bash
DB_HOST=<RDS_ENDPOINT>   # ← 실제 엔드포인트로 교체 안 됨
```

**해결**  
Launch Template v4 → v5: User Data의 `<RDS_ENDPOINT>`를 실제 엔드포인트로 교체 후 재배포.
```bash
DB_HOST=guestbook-db-kih.cr68wwi2oocw.ap-northeast-2.rds.amazonaws.com
```

---

## 최종 IAM 구성 요약

### github-actions-backend-role (GitHub Actions OIDC)
- S3: `PutObject / GetObject` — 아티팩트 버킷
- CodeDeploy: `CreateDeployment / GetDeployment / RegisterApplicationRevision` 등

### codedeploy-service-role
- 관리형: `AWSCodeDeployRole`
- 인라인 `BlueGreenExtraPermissions`: `autoscaling:*` + EC2 실행 관련 권한
- 인라인 `AllowPassBackendEc2Role`: `iam:PassRole` → `backend-ec2-role`

### backend-ec2-role (EC2 인스턴스 프로파일)
- `AmazonEC2RoleforAWSCodeDeploy`
- `AmazonSSMManagedInstanceCore`
