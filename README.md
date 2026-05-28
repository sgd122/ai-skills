# ai-skills

Claude Code 워크플로우를 자동화하는 **커스텀 스킬(Skill)** 과 **훅(Hook)** 모음입니다.

코드 검증, 스킬 유지보수, 워크트리 병합과 같은 반복 작업을 표준화된 절차로 실행할 수 있도록 구성되어 있습니다.

---

## 📁 디렉토리 구조

```
ai-skills/
├── .claude/
│   ├── settings.json              # 훅 등록 (SessionStart, Stop)
│   ├── hooks/
│   │   ├── commit-session.sh      # 세션 종료 시 자동 커밋 + CHANGELOG 갱신
│   │   └── load-recent-changes.sh # 세션 시작 시 최근 커밋/CHANGELOG 컨텍스트 주입
│   └── skills/
│       ├── manage-skills/         # 검증 스킬 생성/업데이트 관리
│       ├── merge-worktree/        # 워크트리 브랜치 스쿼시 머지
│       └── verify-implementation/ # 모든 verify-* 스킬을 순차 실행
└── CLAUDE.md                      # 프로젝트 지침 (스킬 카탈로그)
```

---

## 🚀 사용 방법

### 사전 준비

[Claude Code](https://docs.claude.com/en/docs/claude-code/overview) CLI가 설치되어 있어야 합니다.

```bash
git clone <this-repo>
cd ai-skills
claude  # Claude Code 세션 시작
```

세션이 시작되면 `.claude/settings.json`의 훅이 자동으로 활성화됩니다.

### 스킬 실행

스킬은 슬래시 커맨드 형태로 호출합니다:

```
/verify-implementation         # 등록된 모든 verify-* 스킬 순차 실행
/manage-skills                 # 세션 변경사항을 분석해 검증 스킬을 생성/갱신
/merge-worktree [target-branch] # 현재 워크트리 브랜치를 타겟 브랜치에 스쿼시 머지
```

> 모든 스킬은 `disable-model-invocation: true`로 설정되어 있어 명시적으로 호출해야 실행됩니다.

---

## 🛠 등록된 스킬

| 스킬 | 목적 | 트리거 |
|------|------|--------|
| [`verify-implementation`](.claude/skills/verify-implementation/SKILL.md) | 프로젝트의 모든 `verify-*` 스킬을 순차 실행해 통합 검증 보고서 생성 | 기능 구현 후, PR 생성 전, 코드 리뷰 시 |
| [`manage-skills`](.claude/skills/manage-skills/SKILL.md) | 세션 변경사항을 분석하여 검증 스킬 드리프트 탐지 및 자동 생성/업데이트 | 새 패턴/규칙 도입 후, PR 전 정합성 점검 시 |
| [`merge-worktree`](.claude/skills/merge-worktree/SKILL.md) | 워크트리 브랜치를 타겟 브랜치로 스쿼시 머지하고 구조화된 커밋 메시지 생성 | 워크트리 작업 완료 후 메인 브랜치에 병합할 때 |

### 스킬 생애주기

```
┌─────────────────────┐
│  코드 변경 발생      │
└──────────┬──────────┘
           ↓
┌─────────────────────┐     생성/업데이트     ┌──────────────────────┐
│   manage-skills     │ ──────────────────→  │ verify-<domain> 스킬 │
└─────────────────────┘                       └──────────┬───────────┘
                                                         │
                                                         ↓
┌─────────────────────┐     순차 실행        ┌──────────────────────┐
│ verify-implementation│ ←──────────────────  │  검증 워크플로우      │
└─────────────────────┘                       └──────────────────────┘
```

`manage-skills`는 `verify-*` 스킬의 카탈로그를 자동으로 관리하며, `verify-implementation`은 등록된 스킬을 일괄 실행합니다.

---

## 🔧 자동화 훅

`.claude/settings.json`에 등록된 훅은 모든 Claude Code 세션에서 자동으로 동작합니다.

| 훅 | 시점 | 동작 |
|----|------|------|
| `load-recent-changes.sh` | `SessionStart` | 최근 git 커밋 10개와 `docs/CHANGELOG.md` 마지막 20줄을 컨텍스트로 주입 |
| `commit-session.sh` | `Stop` | 세션 종료 시 변경사항을 스테이징하고 `claude -p`로 생성한 `WIP(scope): ...` 메시지로 자동 커밋. 실패 시 fallback 메시지 사용 |

> `commit-session.sh`는 `claude -p`를 호출하므로 PATH에 `claude` CLI가 있어야 합니다. 없으면 fallback 메시지로 작동합니다.

---

## 🧩 새 검증 스킬 추가하기

새 검증 스킬은 직접 만들지 말고 **`/manage-skills`** 를 사용하세요. 이 스킬은:

1. 현재 세션의 변경 파일을 분석
2. 기존 `verify-*` 스킬과 매핑
3. 커버되지 않은 패턴을 식별
4. 사용자 승인 후 새 스킬 생성 또는 기존 스킬 확장
5. `manage-skills`, `verify-implementation`, `CLAUDE.md`의 카탈로그를 자동 동기화

직접 생성해야 하는 경우 다음 규칙을 따르세요:

- 디렉토리: `.claude/skills/verify-<domain>/SKILL.md`
- 이름은 `verify-` 접두사 + kebab-case
- frontmatter에 `name`, `description`, `disable-model-invocation: true` 포함
- 필수 섹션: `Purpose`, `When to Run`, `Related Files`, `Workflow`, `Output Format`, `Exceptions`

자세한 템플릿은 [`manage-skills/SKILL.md`](.claude/skills/manage-skills/SKILL.md#step-6-새-스킬-생성)를 참고하세요.

---

## 📚 참고

- [Claude Code 공식 문서](https://docs.claude.com/en/docs/claude-code/overview)
- [Skills 개요](https://docs.claude.com/en/docs/claude-code/skills)
- [Hooks 개요](https://docs.claude.com/en/docs/claude-code/hooks)
