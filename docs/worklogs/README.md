# Worklogs

## 목적

작업 기록은 특정 구현 또는 장애를 다시 재현할 때 필요한 세부 근거를 보존한다. 현재 상태의
요약은 `PROJECT_STATUS.md`가 담당하므로 새 채팅에서 worklog 전체를 읽지 않는다.

## 파일 이름

`YYYY-MM-DD-short-topic.md`

같은 날짜에 작업이 여러 개라면 topic을 구분한다.

## 작성 시점

- 재사용 가치가 있는 오류와 해결 과정이 생김
- 장애 실험과 복구 결과를 기록함
- 성능 또는 ML 실험의 조건과 결과를 보존함
- 마일스톤 handoff에 세부 명령과 로그가 필요함

모든 작은 파일 수정에 worklog를 만들지는 않는다.

## Template

```markdown
# 작업 제목

## Metadata

- Date:
- Milestone:
- Status:

## Goal

## Files changed

## Decisions

## Commands and verification

| Command | Result |
|---|---|

## Failures and diagnosis

## Remaining work

## Next recommended action
```
