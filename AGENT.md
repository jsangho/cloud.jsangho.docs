# 코딩 규칙 인덱스

그래프는 **경로**로 프론트·백엔드 클러스터가 나뉜다. 허브 인덱스 파일은 쓰지 않는다.

| 영역 | 코드 경로 | 규칙 문서 |
|------|-----------|-----------|
| 프론트엔드 | `www/` | `www/.cursorrules` · `www/_claude/REACT_RULES.md` |
| 백엔드 | `sangho/` | `sangho/.cursorrules` · `sangho/_claude/ENTITY_RULE.md` · 앱 `_docs/` |

```text
  www/* … (프론트)          sangho/* … (백엔드)
       \                      /
        공통: .cursorrules · CLAUDE.md · agent.md
```

1. 작업 디렉터리에 맞는 `.cursorrules`를 Read한다.
2. 해당 `.cursorrules`가 가리키는 규칙 MD를 Read한다.
