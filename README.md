# Keyport

내 Cloudflare 계정에서 운영하는 Tailscale 기기 등록 키 발급 PWA. 모바일에서 계정과 네트워크를 선택하고, 24시간 유효한 일회용 키를 발급·복사합니다.

[![Cloudflare에 배포](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/ajchemist/keyport)

## 브라우저에서 설치

1. 위 버튼을 누르고 **배포자 자신의 GitHub와 Cloudflare 계정**을 연결합니다. 생성할 저장소·Worker·D1 이름을 확인합니다.
2. 자동으로 채워지는 빌드 명령 `npm run build`, 배포 명령 `npm run deploy`를 확인합니다. Node 24는 저장소의 `.node-version`으로 지정합니다. D1은 자동 생성·바인딩되며 DB ID를 수동으로 복사하지 않습니다. 배포 명령이 초기 마이그레이션도 수행합니다.
3. 배포된 Worker URL을 열면 **설정 wizard**가 나옵니다. 별도 CLI 설치는 필요 없습니다.
4. wizard에 따라 Worker 전체를 Access로 보호하고, 실제 관리자 이메일만 허용합니다. 브라우저에서 암호화 키를 생성하고 복구 파일을 안전하게 보관합니다.
5. wizard가 준비한 다섯 항목을 Worker **Settings → Variables and Secrets**에 **Secret**으로 넣고 Deploy합니다. Access 로그인 후 앱을 엽니다.

자동화되는 범위는 저장소 복제·Workers Builds 연결·D1 생성·바인딩·마이그레이션·앱 배포입니다. **Access 정책과 Secret 저장은 사용자가 wizard를 따라 수행합니다.** Cloudflare 계정 관리 토큰을 앱에 맡기지 않습니다. 공식 배포 버튼이 제공하는 흐름이며, 실제 빈 Cloudflare 계정에서의 종단 검증은 아직 남아 있습니다. [배포 버튼 공식 안내](https://developers.cloudflare.com/workers/platform/deploy-buttons/)

설치 안내만 인증 전 공개됩니다. 계정 등록·토큰 조회·발급 API와 앱 화면은 설정이 끝나기 전 차단됩니다. 설정 완료 후 wizard도 인증이 필요합니다. 첫 방문자를 관리자로 등록하지 않습니다.

## 사용

- 계정 프로필의 표시 이름과 예상 소유자를 입력하고, 해당 프로필에 Tailnet 식별자와 **사용자 API 토큰(`tskey-api-`)**을 등록합니다. OAuth 자격 증명은 받지 않습니다.
- 등록 시 해당 Tailnet의 키 목록 접근을 검증합니다. 예상 소유자는 **관리자 입력·미검증**이며, 키 목록 조회 성공이 실제 발급 권한이나 소유자 신원을 보증하지 않습니다. 토큰 만료 예정일도 관리자 입력입니다.
- 계정 → 네트워크를 선택해 키를 발급하고, 선택한 연결과 예상 소유자·만료 시각을 확인한 뒤 복사해 비공개 경로로 직접 전달합니다.
- 키는 일회용·사전 승인·비임시·태그 없음·24시간으로 서버가 강제합니다. Tailnet Lock이 켜져 있으면 별도 서명이 필요할 수 있습니다. 기존 보안 정책을 끄지 마세요.
- 같은 Tailnet을 다른 프로필과 다른 사용자의 토큰으로 각각 등록할 수 있습니다. 모든 허용 관리자가 모든 연결을 공동 관리합니다.
- 토큰은 저장 후 다시 표시하지 않습니다. 발급 키는 현재 화면 메모리에만 있으며 지우기·다른 연결 선택·화면 이탈·백그라운드 전환·새로고침 때 제거합니다. 복사한 값은 OS 클립보드에 남을 수 있습니다.
- 발급은 자동 재시도하지 않습니다. 타임아웃·불완전 응답·`pending` 기록은 성공 여부를 알 수 없는 상태입니다. Tailscale 콘솔을 확인하고 불필요한 키를 폐기한 뒤 필요할 때 새로 발급하세요.
- 사용자 API 토큰의 만료(최대 90일), 발급 키의 24시간 만료, 등록된 기기의 노드 키 만료는 각각 다릅니다. 토큰은 만료 전에 연결 관리에서 교체합니다. [Tailscale API 인증](https://tailscale.com/docs/reference/tailscale-api), [기기 등록 키](https://tailscale.com/docs/features/access-control/auth-keys)

## 구성과 보호

Astro 정적 페이지 + UnoCSS + TypeScript Worker + D1 하나입니다. 프런트엔드 런타임 프레임워크, KV, R2, Durable Objects, 서비스 워커, 분석 스크립트는 사용하지 않습니다.

토큰은 AES-256-GCM으로 암호화하며 무작위 96비트 nonce와 연결 ID AAD를 사용합니다. 마스터 키는 D1과 분리한 Worker Secret입니다. 고정된 Tailscale API 호스트만 호출하고 리다이렉트도 거부합니다. 토큰·발급 키는 URL이나 브라우저 영구 저장소에 넣지 않습니다. 서버 오류에 upstream 본문을 포함하지 않습니다.

Access의 Worker 전체 보호를 설정하고, 서버도 JWT의 서명·issuer·audience·만료와 관리자 이메일을 검증합니다. 설정한 단일 origin 이외의 도메인은 서버에서 차단합니다. 정적 자산을 쓰면 네이티브 `ctx.access`가 전달되지 않는 제한 때문에 JWT 검증을 사용합니다. `run_worker_first`로 자산 요청도 Worker를 거칩니다. [Access 공식 문서](https://developers.cloudflare.com/workers/configuration/cloudflare-access/)

변경 요청은 Origin·JSON Content-Type·전용 헤더를 검사하며 외부 CORS는 열지 않습니다. D1 원자적 갱신으로 관리자별 발급 간격 5초, 연결별 30초를 적용합니다. 모든 응답은 `no-store`이며 CDN 캐시도 금지합니다. 오프라인 발급·비밀 요청 큐·재실행 시 과거 키 복원은 없습니다.

## 업데이트와 운영

[운영 안내](docs/OPERATIONS.md)에 업데이트·롤백·토큰 갱신·백업/복구·암호화 키 교체·리소스 삭제 절차가 있습니다. [검증 기록](docs/VERIFICATION.md)은 자동 테스트와 실제 환경의 미검증 항목을 구분합니다. 도메인 용어는 [CONTEXT.md](CONTEXT.md), 결정 배경은 [ADR](docs/adr/)에 기록합니다.

이 저장소의 변경이 배포자 복제 저장소에 자동 병합되지는 않습니다. 배포자 저장소의 연결된 운영 브랜치에 변경을 반영하면 Workers Builds가 재배포합니다. 기존 D1 바인딩 ID와 Secret을 유지하세요.

## 로컬 개발

Node 24 이상:

```sh
npm ci
npm run types
npm run check
npm test
npm run build
npm run db:local
npm run dev
```

기본 로컬 실행은 미설정 상태로 wizard만 제공합니다. 운영에 인증 우회 개발 모드는 없습니다. 테스트는 합성 JWT·임시 SQLite·모의 Tailscale API로 검증합니다. 실제 사용자 토큰을 테스트 파일이나 채팅에 넣지 마세요.

## 비용

2026-10-19 확인 기준, 작은 관리자 인스턴스는 Workers Free·D1 Free·Access Free의 범위에서 시작할 수 있습니다. 무료 운영을 보장하지는 않습니다.

| 리소스 | 무료 범위 / 비용 조건 |
| --- | --- |
| Workers | 하루 100,000 요청, 호출당 CPU 10ms. 모든 정적 요청도 인증 Worker를 거칩니다. 한도 또는 CPU 예산을 넘으면 서비스 제한/실패가 가능하며 Paid 전환 시 기본 구독·초과 사용료가 생깁니다. |
| D1 | 하루 5백만 행 읽기·10만 행 쓰기·총 5GB. 무료 한도를 넘으면 작업이 제한됩니다. Paid는 포함량 초과 사용량에 과금합니다. |
| Access | 소규모 팀용 무료 플랜. 50명 이상 규모·추가 기능은 플랜을 확인하세요. |
| Builds | 빌드 시간·동시성·월 한도는 Workers Builds 플랜을 따릅니다. |

[Workers 요금](https://developers.cloudflare.com/workers/platform/pricing/), [D1 요금](https://developers.cloudflare.com/d1/platform/pricing/), [Access 요금](https://www.cloudflare.com/sase/products/access/), [Builds 한도와 요금](https://developers.cloudflare.com/workers/ci-cd/builds/limits-and-pricing/). 도메인·IdP·Tailscale 요금은 별도이며 계정 전체의 다른 사용량도 포함해 확인하세요.

## 라이선스와 참고

MIT © 2026 ajchemist. [tailscale-x/tailscale-cloudflare](https://github.com/tailscale-x/tailscale-cloudflare)를 확인했으나 현재 저장소는 다른 구성의 프로젝트이며 라이선스를 확인할 수 없어 코드를 재사용하지 않았습니다. 기존 비공개 onboarding 프로젝트의 코드·비밀도 포함하지 않습니다.
