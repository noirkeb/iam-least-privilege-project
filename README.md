# IAM 최소 권한 정책 설계: 개발/운영 환경 분리

## 개요

스타트업 환경에서 개발자가 실수로 운영(production) 데이터베이스를 삭제하거나 수정하는 사고를 방지하기 위해, AWS IAM 정책을 사용하여 개발팀의 권한 범위를 제한하는 프로젝트입니다.

- 개발팀은 `dev-` 접두사가 붙은 RDS 리소스에 대해 전체 권한을 가짐
- `prod-` 접두사가 붙은 운영 리소스는 삭제, 수정, 중지, 재부팅 작업을 명시적으로 차단
- 모든 리소스에 대한 조회(Describe) 권한은 허용하여 업무 가시성 확보

## 문제 상황

개발팀 전체에 AWS 관리자 권한(AdministratorAccess)을 부여하는 것은 다음과 같은 위험이 있습니다.

- 실수로 운영 DB를 삭제하거나 중지시킬 가능성
- 최소 권한 원칙(Principle of Least Privilege) 위반
- 사고 발생 시 책임 소재 파악이 어려움

## 아키텍처 / 정책 구조

정책은 세 개의 Statement로 구성됩니다.

1. **AllowFullAccessToDevResources**: `dev-*` 리소스에 대해 모든 RDS 액션 허용
2. **AllowListAllDBInstances**: 모든 리소스에 대해 조회(Describe, ListTags) 허용
3. **DenyAllActionsOnProdResources**: `prod-*` 리소스에 대해 삭제/수정/중지/재부팅 명시적 차단

AWS IAM 평가 로직상 **명시적 Deny는 다른 Allow보다 항상 우선 적용**되므로, 향후 다른 정책이 실수로 prod 리소스에 대한 권한을 허용하더라도 이 Deny 규칙이 안전장치 역할을 합니다.

## 정책 JSON

`policies/developer-policy.json` 파일 참고

## 구현 과정

1. IAM 콘솔에서 위 JSON으로 관리형 정책(`DeveloperLeastPrivilegePolicy`) 생성
2. `Developers` 그룹 생성 후 해당 정책 연결
3. 테스트용 사용자(`test-developer`)를 그룹에 추가
4. IAM Policy Simulator로 정책 동작 검증

## 검증 결과

IAM Policy Simulator를 사용하여 실제 리소스를 생성하지 않고도 정책이 의도대로 작동하는지 확인했습니다.

| 테스트 케이스 | 액션 | 대상 리소스 | 예상 결과 | 실제 결과 |
|---|---|---|---|---|
| 개발 리소스 삭제 | DeleteDBInstance | `dev-test-01` | Allowed | ✅ Allowed |
| 운영 리소스 삭제 | DeleteDBInstance | `prod-main-01` | Denied | ✅ Denied |

관련 스크린샷은 `screenshots/` 폴더에 있습니다.

## 배운 점

- IAM 정책 평가 시 명시적 Deny가 Allow보다 우선한다는 점을 실제로 시뮬레이션하며 확인함
- 리소스 이름에 네이밍 컨벤션(`dev-`, `prod-` 접두사)을 적용하면 ARN 패턴 매칭만으로 환경별 권한 분리가 가능하다는 것을 학습함
- 실제 리소스를 배포하지 않고도 Policy Simulator로 안전하게 정책을 검증할 수 있음

## 사용 기술

- AWS IAM (Identity and Access Management)
- AWS IAM Policy Simulator
- AWS RDS (정책 대상 서비스)
