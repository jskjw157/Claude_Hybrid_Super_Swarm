# Claude Code Agent Farm - macOS 트러블슈팅 가이드

이 문서는 macOS에서 Claude Code Agent Farm 설치 및 실행 시 발생할 수 있는 문제와 해결 방법을 정리한 것입니다.

## 환경 정보

- **OS**: macOS (Darwin 24.6.0, Apple Silicon)
- **Shell**: zsh (macOS 기본)
- **Claude Code**: v2.0.76
- **Python**: 3.13+

---

## 문제 1: `cc` alias가 zsh에서 작동하지 않음

### 증상
```
Agent 00: Claude Code failed to start properly after 5 attempts
```

### 원인
`setup.sh`가 `cc` alias를 `~/.bashrc`에만 추가하고 `~/.zshrc`에는 추가하지 않음. macOS Catalina 이후 기본 쉘이 zsh로 변경됨.

### 해결 방법
```bash
# ~/.zshrc에 alias 추가
echo 'alias cc="ENABLE_BACKGROUND_TASKS=1 claude --dangerously-skip-permissions"' >> ~/.zshrc
source ~/.zshrc
```

---

## 문제 2: tmux에서 `cc` 명령어가 C 컴파일러(clang)로 실행됨

### 증상
alias를 추가했는데도 여전히 `Claude Code failed to start properly` 에러 발생

### 원인
- tmux는 non-interactive shell로 시작되어 `~/.zshrc`의 alias가 로드되지 않음
- macOS의 `/usr/bin/cc`는 C 컴파일러(Apple clang)
- Shell alias보다 시스템 PATH의 `cc`가 우선 실행됨

### 진단 방법
```bash
# tmux 내부에서 cc 실행 테스트
tmux new-session -d -s test_cc "cc --version > /tmp/cc_test.txt 2>&1; exit"
sleep 3
cat /tmp/cc_test.txt

# 만약 "Apple clang version..."이 출력되면 C 컴파일러가 실행된 것
```

### 해결 방법
alias 대신 실행 파일을 PATH에 추가:

```bash
# 1. ~/bin 디렉토리에 cc 스크립트 생성
mkdir -p ~/bin
cat > ~/bin/cc << 'EOF'
#!/bin/zsh
ENABLE_BACKGROUND_TASKS=1 claude --dangerously-skip-permissions "$@"
EOF
chmod +x ~/bin/cc

# 2. PATH에 ~/bin 추가 (zshrc와 zprofile 모두)
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zprofile

# 3. 새 터미널 열기 또는 source
source ~/.zshrc
```

### 왜 zprofile에도 추가하는가?
- `~/.zshrc`: interactive shell에서 로드
- `~/.zprofile`: login shell에서 로드 (tmux가 사용)

---

## 문제 3: Claude Code v2.0.76 UI 변경으로 Agent Farm이 시작 감지 실패

### 증상
`cc` 명령어가 정상 실행되지만 여전히 `failed to start properly` 에러

### 원인
Claude Code v2.0.76에서 UI가 변경되어 Agent Farm의 `is_claude_ready()` 함수가 새 패턴을 인식하지 못함.

| Agent Farm이 찾는 패턴 | 실제 출력 (v2.0.76) |
|------------------------|---------------------|
| `"Welcome to Claude Code!"` | `"Welcome back [username]!"` |
| `"│ > Try"` | `"> Try"` |
| `"? for shortcuts"` | `"bypass permissions on"` |

### 해결 방법
`claude_code_agent_farm.py`의 `is_claude_ready()` 함수 수정:

```python
def is_claude_ready(self, content: str) -> bool:
    """Check if Claude Code is ready for input"""
    ready_indicators = [
        "Welcome to Claude Code!" in content,  # 기존 환영 메시지
        "Welcome back" in content,  # 새 환영 메시지 (v2.0.76+)
        ("│ > Try" in content),  # 기존 프롬프트
        ("> Try" in content),  # 새 프롬프트 형식 (v2.0.76+)
        ("? for shortcuts" in content),  # 기존 힌트
        ("bypass permissions" in content),  # 새 바이패스 힌트 (v2.0.76+)
        ("╰─" in content and "│ >" in content),
        ("/help for help" in content),
        ("cwd:" in content and "Welcome to Claude" in content),
        ("Bypassing Permissions" in content and "│ >" in content),
        ("│ >" in content and "─╯" in content),
        ("Claude Code v" in content),  # 버전 헤더 (v2.0.76+)
    ]
    return any(ready_indicators)
```

---

## 문제 4: 프로젝트 설정 문제 (Next.js)

### 증상
type-check나 lint 명령어가 실패하거나 예상치 못한 에러 발생

### 해결 체크리스트

#### 4.1 npm 패키지 설치
```bash
cd /path/to/your/project
npm install
```

#### 4.2 tsconfig.json - reference 폴더 제외
불필요한 폴더(예: reference, examples)를 exclude에 추가:
```json
{
  "exclude": ["node_modules", "reference"]
}
```

#### 4.3 tsconfig.json - Vitest globals 타입 추가
Vitest 사용 시 globals 타입 인식을 위해:
```json
{
  "compilerOptions": {
    "types": ["vitest/globals"],
    ...
  }
}
```

#### 4.4 커스텀 config 생성
프로젝트에 맞는 config 파일 생성 (bun 대신 npm 사용 등):

```json
{
  "comment": "Configuration for Your Project",
  "tech_stack": "nextjs",
  "problem_commands": {
    "type_check": ["npx", "tsc", "--noEmit"],
    "lint": ["npm", "run", "lint"]
  },
  "agents": 5,
  "session": "your_project_agents",
  "skip_commit": true,
  "auto_restart": true
}
```

---

## 전체 설치 체크리스트

```bash
# 1. 레포지토리 클론
git clone https://github.com/Dicklesworthstone/claude_code_agent_farm.git
cd claude_code_agent_farm

# 2. setup.sh 실행
chmod +x setup.sh
./setup.sh

# 3. zsh용 cc 스크립트 생성 (macOS 필수)
mkdir -p ~/bin
cat > ~/bin/cc << 'EOF'
#!/bin/zsh
ENABLE_BACKGROUND_TASKS=1 claude --dangerously-skip-permissions "$@"
EOF
chmod +x ~/bin/cc

# 4. PATH 설정
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zprofile

# 5. 새 터미널 열기

# 6. 테스트
tmux new-session -d -s test "cc --version > /tmp/test.txt; exit"
sleep 5
cat /tmp/test.txt  # "Claude Code" 버전이 출력되어야 함

# 7. 실행
cd claude_code_agent_farm
source .venv/bin/activate
claude-code-agent-farm --path /your/project --config configs/your_config.json --agents 5
```

---

## 유용한 디버깅 명령어

```bash
# tmux 세션 목록 확인
tmux list-sessions

# 에이전트 상태 확인
cat /path/to/project/.claude_agent_farm_state.json | python3 -m json.tool

# 특정 에이전트 화면 캡처
tmux capture-pane -t session_name:1.0 -p | tail -50

# 문제 파일 확인
cat /path/to/project/combined_typechecker_and_linter_problems.txt | head -30

# tmux 세션 강제 종료
tmux kill-session -t session_name
```

---

## 참고 사항

- Agent Farm은 tmux 내에서 여러 Claude Code 인스턴스를 병렬로 실행합니다
- 각 에이전트는 약 500MB RAM을 사용합니다
- `Ctrl+C`로 graceful shutdown, 3초 내 다시 `Ctrl+C`로 강제 종료
- 작업 완료 후 HTML 리포트가 프로젝트 디렉토리에 생성됩니다

---

*Last Updated: 2026-01-14*
*Claude Code Version: 2.0.76*
*Agent Farm Repository: https://github.com/Dicklesworthstone/claude_code_agent_farm*
