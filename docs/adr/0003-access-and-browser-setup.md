# 브라우저 wizard로 Access와 Secret 설정을 마무리한다

Deploy to Cloudflare로 저장소·리소스·배포를 준비하고, 공개 설치 안내 wizard에서 배포자가 Access 정책과 Secret을 설정한다. 인증 전 비밀 등록이나 최초 방문자 관리자 지정은 하지 않는다. 계정 관리 권한을 런타임에 보관하는 자동 설정 대신 사용자가 대시보드에서 수행하는 단계를 남겨 권한 범위를 줄였다. 필수 설정 전에는 관리 UI와 API를 차단하며, 완료 후에는 설치 안내도 인증을 요구한다.

정적 자산 Worker에는 네이티브 ctx.access가 전달되지 않는 현재 플랫폼 제한 때문에 JWT 서명·issuer·audience·만료와 명시적 관리자 목록을 서버에서 검증한다. Worker 전체 Access 보호에 더해 단일 origin 검사를 적용해 대체 도메인과 프리뷰를 차단한다. 화면은 Astro 정적 출력과 UnoCSS, 브라우저 동작은 작은 ES 모듈로 구성한다.
