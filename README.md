# 약학다식 · PillGuide GitOps

**약학다식**의 EC2 / K3s 클러스터에 적용할 Kubernetes 리소스를 관리합니다. 애플리케이션 소스 저장소가 아니라, Argo CD가 배포 상태의 기준으로 삼는 **선언형 설정 저장소**입니다.

## 배포 아키텍처

```text
Backend main / develop
        ↓ GitHub Actions: 검증·빌드·ARM64 이미지 발행
manifest/main: 환경별 Deployment 이미지 digest 갱신
        ↓ Argo CD 자동 동기화
EC2 / K3s: production · dev 네임스페이스
        ├─ Spring Pod / Service / Ingress
        ├─ External Secrets → 실행 시 Secret 공급
        └─ 공통 CoreDNS 설정 → Tailscale 내 로컬 AI 서버 이름 해석
```

| Backend 브랜치 | 이 저장소의 경로 | 네임스페이스 |
|---|---|---|
| `develop` | `dev/` | `dev` |
| `main` | `production/` | `production` |

두 환경 모두 **이 저장소의 `main`**을 추적합니다. 환경별 Git 브랜치를 나누지 않습니다. Backend CI는 대상 환경의 이미지 digest만 바꾸고, replica 등 배포 정책은 이 저장소에서 관리합니다.

## 저장소 구조

| 경로 | 역할 |
|---|---|
| [argocd/root.yaml](argocd/root.yaml) | 하위 Application을 관리하는 진입점 |
| [argocd/apps/](argocd/apps/) | dev·production·플랫폼·External Secrets Application |
| [dev/](dev/), [production/](production/) | Deployment, Service, Ingress, 설정·Secret 참조, 자원 제한 |
| [platform/argocd/](platform/argocd/) | Argo CD 접근용 Ingress |
| [platform/coredns/](platform/coredns/) | 두 환경이 공유하는 추론 서버 DNS 설정 |
| [templates/](templates/) | 배포 참고 템플릿; 현재 Application의 배포 소스가 아님 |

## 운영 원칙

- `main` 변경은 자동 동기화로 실제 클러스터에 반영될 수 있습니다. 반영 전에 환경과 diff를 검토합니다.
- `selfHeal`이 켜져 있으므로 수동 변경은 Git의 값으로 복구될 수 있습니다. replica 조정과 롤백도 Git 변경으로 수행합니다.
- 현재 dev·production은 각각 **1 replica / Recreate**로, 배포 중 짧은 중단을 허용합니다.
- 키·비밀번호는 Secret 관리 경로로 공급합니다. ConfigMap이나 README에 실제 비밀값을 기록하지 않습니다.
- Git 기반 Application은 `main`을 추적하고 External Secrets Helm 차트는 명시된 버전에 고정합니다.

### 공유 DNS 주의사항

`platform-dns`는 K3s의 custom CoreDNS import를 이용하며 관리형 Corefile을 교체하지 않습니다. 추론 서버의 사설 주소가 바뀌면 매핑을 검토해야 합니다. 공유 ConfigMap을 dev·production에 중복 선언하지 않습니다.

이 Application은 자동 동기화·self-heal을 사용하지만 **자동 prune은 비활성화**돼 있고 cascade finalizer도 없습니다. Git에서 정의나 Application을 없애도 DNS ConfigMap이 남을 수 있으므로 삭제는 별도 검토가 필요합니다. Tailscale 연결·접근 정책, 데스크톱 서버 실행·자동 시작, API 키는 이 저장소가 설정하지 않습니다.

## 적용·검증

K3s·Argo CD·저장소 접근 권한과 Secret 공급 경로가 준비된 승인 환경에서 [root Application](argocd/root.yaml)을 bootstrap합니다. 첫 환경 동기화 전 External Secrets의 준비 상태를 확인합니다.

검증은 다음 두 단계로 나눕니다.

1. `pillguide-root`, `argocd-config`, `external-secrets`, `platform-dns`, `dev`, `production`의 동기화·상태와 Backend readiness 확인.
2. 실제 dev·production Backend 컨테이너에서 DNS, TLS 검증을 포함한 인증된 AI `/health`·`/predict`, 기존 서비스·DB 연결 확인.

**Synced/Healthy 또는 sync wave만으로 실제 통신 성공을 보장하지 않습니다.** 인프라 생성·클러스터 설치·데스크톱 추론 서버 배포는 이 저장소의 범위 밖입니다.

## 연결 문서

- [Backend — 이미지 빌드와 GitOps 전달 과정](https://github.com/pillguide/Backend/blob/develop/docs/gitops.md)
- [Frontend — Vercel 환경별 라우팅](https://github.com/pillguide/Frontend/blob/main/docs/deployment.md)
- [AI-server — 추론 API 연동 명세](https://github.com/pillguide/AI-server/blob/main/docs/SPRING_INTEGRATION.md)
- [Argo CD 자동 동기화](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/) · [K3s custom CoreDNS](https://docs.k3s.io/advanced#coredns-custom-configuration-imports)
