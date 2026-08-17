# claude-global-config

Claude Code 전역 설정(`~/.claude/CLAUDE.md`, `~/.claude/settings.json`, 재사용 가능한
`agents/`, `skills/`) 백업/버전관리 저장소.

## 새 기기/새 환경에서 적용하는 법

```bash
git clone https://github.com/izumilover/claude-global-config ~/claude-global-config
ln -sf ~/claude-global-config/CLAUDE.md ~/.claude/CLAUDE.md
ln -sf ~/claude-global-config/CLAUDE_spring.md ~/.claude/CLAUDE_spring.md
ln -sf ~/claude-global-config/settings.json ~/.claude/settings.json
ln -sf ~/claude-global-config/agents ~/.claude/agents
mkdir -p ~/.claude/skills
for d in ~/claude-global-config/skills/*/; do
  ln -sf "${d%/}" ~/.claude/skills/"$(basename "$d")"
done
```

심볼릭 링크로 연결해두면, 이후 `~/.claude/CLAUDE.md`를 수정해도 실제로는 이 저장소 안의 파일이
바뀌는 것이라 `git add && git commit && git push`로 바로 버전관리 및 다른 기기 동기화가 가능하다.

## bkit 플러그인

`settings.json`에 bkit 마켓플레이스·플러그인 활성화가 이미 선언되어 있어서(`extraKnownMarketplaces`,
`enabledPlugins`), 위 symlink만 걸면 Claude Code가 다음 실행 시 자동으로 인식·설치를 시도한다.

혹시 자동으로 안 잡히면, Claude Code 세션 안에서 아래 명령을 순서대로 실행한다:

```
/plugin marketplace add popup-studio-ai/bkit-claude-code
/plugin install bkit@bkit-marketplace
```

## agents / skills

`agents/spring-*.md` + `skills/spring-webapp`, `skills/spring-security-checklist`,
`skills/thymeleaf-patterns`는 Java/Spring Boot 웹앱 개발용 범용 하네스(에이전트 팀 5명 +
오케스트레이터 스킬 1개 + 확장 스킬 2개)다. 특정 프로젝트에 종속되지 않고 어떤 Spring Boot
프로젝트에서든 "Spring 웹앱 만들어줘" 등으로 트리거해서 재사용한다. 개요는 `CLAUDE_spring.md`,
상세 워크플로우는 `skills/spring-webapp/SKILL.md` 참고. `CLAUDE.md`에 "Spring 프로젝트면
`CLAUDE_spring.md`를 먼저 읽는다"는 지침이 걸려 있어 자동으로 안내된다.

## 주의

- `~/.claude/` 전체를 넣지 않는다 — `.credentials.json`, `history.jsonl`, 프로젝트별 메모리
  (`projects/`) 등 민감하거나 기기·계정에 종속적인 파일이 섞여 있다.
- 여기 들어가는 파일은 항상 "다른 프로젝트/다른 기기에 그대로 재사용해도 되는" 내용인지
  확인한 뒤 추가한다.
