# Troubleshooting Log

이 디렉터리는 프로젝트 진행 중 발생한 오류와 해결 과정을 기록한다.

각 이슈는 아래 형식을 기준으로 정리한다.

## 기록 원칙

1. 오류 메시지를 원문 그대로 남긴다.
2. 발생 시점의 환경과 실행 명령을 함께 기록한다.
3. 추정 원인과 실제 원인을 구분한다.
4. 최종 해결 명령을 재현 가능하게 남긴다.
5. 해결 후 검증 명령과 정상 결과를 남긴다.
6. 비밀번호, 토큰, 인증서 키, 개인키 등 민감정보는 기록하지 않는다.

## 권장 템플릿

```markdown
# <Issue title>

## Environment
- Node:
- OS:
- Kubernetes:
- Runtime:
- Component:

## Symptom
```text
<error message>
```

## Command
```bash
<command that triggered the issue>
```

## Initial hypothesis
- 

## Root cause
- 

## Resolution
```bash
<actual fix commands>
```

## Verification
```bash
<verification commands>
```

Expected / observed result:
```text
<result>
```

## Lesson learned
- 

## Prevention
- 
```

## Current issues

- `cilium-helm-repo-not-found.md` — Helm repository 미등록으로 `cilium/cilium` 차트를 찾지 못한 사례

향후 구축 중 발생하는 오류는 동일한 형식으로 계속 추가한다.
