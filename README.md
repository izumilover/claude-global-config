# claude-global-config

Claude Code 전역 설정(`~/.claude/CLAUDE.md`, `~/.claude/settings.json`) 백업/버전관리 저장소.

## 새 기기/새 환경에서 적용하는 법

```bash
git clone <이 저장소 URL> ~/claude-global-config
ln -sf ~/claude-global-config/CLAUDE.md ~/.claude/CLAUDE.md
ln -sf ~/claude-global-config/settings.json ~/.claude/settings.json
```

심볼릭 링크로 연결해두면, 이후 `~/.claude/CLAUDE.md`를 수정해도 실제로는 이 저장소 안의 파일이
바뀌는 것이라 `git add && git commit && git push`로 바로 버전관리 및 다른 기기 동기화가 가능하다.

## 주의

- `~/.claude/` 전체를 넣지 않는다 — `.credentials.json`, `history.jsonl`, 프로젝트별 메모리
  (`projects/`) 등 민감하거나 기기·계정에 종속적인 파일이 섞여 있다.
- 여기 들어가는 파일은 항상 "다른 프로젝트/다른 기기에 그대로 재사용해도 되는" 내용인지
  확인한 뒤 추가한다.
