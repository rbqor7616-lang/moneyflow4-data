# moneyflow4-data

머니플로우 앱이 사용하는 시장 데이터 스냅샷 저장소입니다. 저장소에는 `snapshot.json` 한 파일만 있으며,
지수·자산·섹터·금리·환율·공포탐욕지수·뉴스 등이 한 줄짜리 압축 JSON으로 들어 있습니다.

외부 앱이 매일 21:53(UTC)에 시세와 공포탐욕지수를 갱신해 push 하지만, `news` 블록은 갱신되지 않고 멈춰 있었습니다.
이 저장소의 GitHub Actions 워크플로가 그 부분을 대신 채워 줍니다.

## 자동 갱신 워크플로

- 워크플로: `.github/workflows/refresh-news.yml` (이름: **Refresh news**)
- 실행 주기: 매시 20분(UTC) + 수동 실행
- 하는 일:
  1. Google News RSS(미국 증시 검색 피드)에서 최신 기사를 가져옵니다.
  2. 4일이 지난 기사는 버리고, 링크·제목 기준으로 중복을 제거한 뒤 최신순 15건을 남깁니다.
  3. `ANTHROPIC_API_KEY`가 있으면 Claude API로 한 번에 번역·분류(호재/악재/중립, 시장/기업, 단기/장기, 원인)합니다.
     키가 없거나 호출이 실패하면 키워드 기반 자동 분류로 대체하고 `news.analyzed`를 `false`로 둡니다.
  4. `snapshot.json`의 `news` 블록만 교체하고, 외부 작성기 버그로 중복되는 `fearGreedHistory`의 같은 날짜 항목을 정리합니다.
     최상위 `updatedAt`을 포함한 다른 키는 건드리지 않습니다.
  5. 변경이 있으면 `github-actions[bot]` 이름으로 커밋하고 push 합니다.

`snapshot.json`은 한 줄짜리 JSON이라 git이 자동 병합할 수 없습니다. 그래서 rebase 대신
**fetch → reset → 갱신 → push** 를 최대 3회 반복합니다.

1. `git fetch origin $BRANCH` 후 `git reset --hard origin/$BRANCH` 로 항상 최신 tip에서 시작합니다.
2. 그 위에서 `scripts/refresh_news.py` 를 실행합니다. 스크립트가 `news`/`fearGreedHistory`만 다시 쓰므로
   외부 writer가 넣은 시세는 그대로 유지됩니다(= 스크립트 자체가 병합기 역할).
3. `snapshot.json`에 변경이 없으면 `no change` 를 출력하고 종료합니다.
4. 변경이 있으면 커밋 후 `git push origin HEAD:$BRANCH`. push가 거절되면(외부 writer가 그 사이에 push) 대기 후
   1번부터 다시 실행합니다. 3회 모두 실패하면 job 이 실패합니다.

매시간 돌지만 헤드라인이 그대로면 커밋을 만들지 않습니다. 새로 받은 기사 링크 목록이 기존 `news.items`와 동일하면
Anthropic 호출 전에 바로 종료하며(토큰 소모 없음), `news.updatedAt`도 건드리지 않습니다.
이때 `fearGreedHistory`에 중복 날짜가 남아 있으면 그 정리만 반영해 커밋합니다.
단, 기존 `news.analyzed`가 `false`이고 API 키가 있으면 키워드 분류 결과를 LLM 분류로 개선하기 위해 그대로 진행합니다.

RSS에서 5건 미만만 확보되면 기존 뉴스를 그대로 두고 아무것도 쓰지 않습니다.

## ANTHROPIC_API_KEY 설정 (선택)

한국어 번역·분류 품질을 높이려면 Anthropic API 키를 저장소 시크릿으로 등록하세요.

1. GitHub 저장소 → **Settings** → **Secrets and variables** → **Actions**
2. **New repository secret** 클릭
3. Name: `ANTHROPIC_API_KEY`, Secret: `sk-ant-...` 키 값 입력 후 저장

키를 등록하지 않아도 워크플로는 정상 동작하며, 이때는 영어 제목 + 키워드 기반 분류가 사용됩니다.

## 브랜치 주의사항

GitHub Actions의 `schedule` 트리거는 **기본 브랜치(main)에 있는 워크플로만** 실행합니다.
따라서 이 워크플로가 담긴 PR을 main에 머지하기 전까지는 매시간 자동 실행이 시작되지 않습니다.
머지한 뒤에는 아래 방법으로 수동 실행해 동작을 먼저 확인할 수 있습니다.
(cron 은 부하에 따라 몇 분 늦게 실행될 수 있습니다.)

## 수동 실행

- GitHub 웹: **Actions** 탭 → 왼쪽에서 **Refresh news** 선택 → **Run workflow**
- gh CLI: `gh workflow run "Refresh news"`

## 로컬 실행

```bash
python3 scripts/refresh_news.py --dry-run          # 결과만 출력, 파일은 쓰지 않음
python3 scripts/refresh_news.py                    # snapshot.json 갱신
python3 scripts/refresh_news.py --input-rss fixture.xml   # 네트워크 없이 로컬 RSS로 테스트
```
