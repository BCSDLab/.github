# BCSDLab/.github

BCSDLab 조직의 여러 레포에서 공통으로 쓰는 GitHub Actions 재사용 워크플로우와 설정 데이터를 모아둔 레포입니다.

## 구조

```
.github/workflows/   # workflow_call 재사용 워크플로우 (로직)
config/<track>/      # 트랙(frontend, backend, ...)별 설정 데이터
```

로직과 데이터를 분리해서, 사람이 바뀌거나(`config/*/reviewer.json`) 알림 방식이 바뀌어도(`.github/workflows/*.yml`) 각 프로젝트 레포를 건드리지 않아도 되도록 합니다.

## Pick Reviewer

PR이 열리면 지정된 트랙의 명단(`config/<track>/reviewer.json`)에서 PR 작성자를 제외하고 랜덤으로 리뷰어를 뽑아 GitHub 리뷰어로 지정하고, 내부 Slack 알림 API를 호출합니다.

### 사용하는 레포 쪽 설정

```yaml
# .github/workflows/pick-reviewer.yml (호출하는 레포)
name: Pick Reviewer

on:
  pull_request:
    types: opened

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}
  cancel-in-progress: false

jobs:
  pick-reviewer:
    uses: BCSDLab/.github/.github/workflows/pick-reviewer.yml@v1
    with:
      track: frontend
      slack-endpoint: frontend
    secrets: inherit
```

리뷰어 선정이 필요 없는 레포는 이 워크플로우 파일을 두지 않으면 됩니다.

### 명단 갱신

`config/<track>/reviewer.json`만 수정하면 해당 트랙을 쓰는 모든 레포에 즉시 반영됩니다.

```json
{
  "reviewers": [
    { "name": "이름", "githubName": "github-id" }
  ]
}
```

## Notify Deploy

`build-and-deploy` job 완료 후 배포자 이름을 명단에서 찾아 내부 Slack 알림 API로 배포 결과를 전송합니다. 실제 빌드/배포(env, SSH, 배포 스크립트 경로 등)는 레포마다 인프라가 달라 여기서 다루지 않고, 각 레포의 `deploy.yml`에 그대로 둡니다.

### 사용하는 레포 쪽 설정

```yaml
# deploy.yml 의 notify job
  notify:
    needs: [build-and-deploy]
    if: always()
    uses: BCSDLab/.github/.github/workflows/notify-deploy.yml@v1
    with:
      track: frontend
      slack-endpoint: frontend
      status: ${{ needs.build-and-deploy.result }}
      environment: ${{ github.ref_name == 'main' && 'Production' || 'Stage' }}
```

### 버전

호출하는 레포는 `@v1` 같은 태그로 고정합니다. 이 레포에서 동작이 바뀌면(예: input 추가/삭제 같은 breaking change) 태그를 올리고, 기존 태그를 쓰는 레포는 영향받지 않습니다.
