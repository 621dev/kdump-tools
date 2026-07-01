# kdump-tools

`kdump-tools`는 Linux `vmcore` 분석을 조금 더 빠르게 시작하기 위한 보조 도구 모음입니다.

현재는 `vmcore`에서 커널 버전을 확인한 뒤, 해당 버전에 맞는 `vmlinux` 디버그 심볼을 찾아 `crash`를 실행하는 `analyze-kdump` 스크립트를 제공합니다.

## 주요 기능

- `vmcore` 파일의 커널 버전 자동 확인
- 커널 버전에 맞는 `vmlinux` 경로 자동 매칭
- 매칭되는 `vmlinux`가 없을 때 필요한 설치 명령 안내
- `crash <vmlinux> <vmcore>` 실행 자동화
- `-report` 옵션으로 셸 진입 없이 요약 리포트 파일 생성
- 선택적으로 LLM을 연동해 요약 리포트에 원인분석과 조치 권고 추가

## 디렉터리 구조

권장 설치 위치는 `/opt/kdump-tools`입니다.

```text
/opt/kdump-tools/
├── README.md
├── analyze-kdump.conf.sample
├── bin/
│   └── analyze-kdump
├── logs/
└── vmlinux/
    └── <kernel-version>/
        └── vmlinux
```

현재 저장소에는 스크립트와 기본 디렉터리만 포함되어 있습니다. 실제 `vmlinux` 파일은 크기가 크고 시스템마다 필요 버전이 다르므로, 분석 대상 커널 버전에 맞춰 별도로 준비합니다.

## 요구 사항

- `bash`
- `crash`
- 분석 대상 `vmcore`
- `vmcore`의 커널 버전과 일치하는 `vmlinux`

RHEL 계열 시스템에서는 보통 `crash`와 `debuginfo-install`을 사용할 수 있어야 합니다.

```bash
dnf install -y crash dnf-plugins-core
```

## 사용법

```bash
/opt/kdump-tools/bin/analyze-kdump /path/to/vmcore
```

경로를 매번 입력하지 않으려면 `/usr/sbin`에 심볼릭 링크를 만들 수 있습니다.

```bash
ln -s /opt/kdump-tools/bin/analyze-kdump /usr/sbin/analyze-kdump
```

이후에는 명령어만으로 실행할 수 있습니다.

```bash
analyze-kdump /path/to/vmcore
```

요약 리포트만 생성하려면 `-report` 또는 `-r` 옵션을 사용합니다.

```bash
/opt/kdump-tools/bin/analyze-kdump -report /path/to/vmcore
/opt/kdump-tools/bin/analyze-kdump -r /path/to/vmcore
```

생성된 리포트를 LLM으로 분석해 원인분석 섹션까지 추가하려면 `-llm` 옵션을 함께 사용합니다. `-llm`은 자동으로 리포트 모드를 켭니다.

```bash
LLM_API_KEY="<api-key>" /opt/kdump-tools/bin/analyze-kdump -llm /path/to/vmcore
```

기본 설정 파일은 `/opt/kdump-tools/analyze-kdump.conf`입니다. 저장소에는 샘플 파일인 `analyze-kdump.conf.sample`만 포함하고, 실제 설정 파일인 `analyze-kdump.conf`는 Git에 올리지 않습니다. 다른 설정 파일을 사용하려면 `-config` 옵션으로 지정합니다.

```bash
/opt/kdump-tools/bin/analyze-kdump -config /path/to/analyze-kdump.conf -report /path/to/vmcore
```

conf 파일에서 `DEFAULT_VMCORE`를 지정하면 명령줄에서 vmcore 경로를 생략할 수 있습니다.

```bash
/opt/kdump-tools/bin/analyze-kdump -report
```

스크립트는 다음 순서로 동작합니다.

1. `crash --osrelease <vmcore>`로 `vmcore`의 커널 버전을 확인합니다.
2. `/opt/kdump-tools/vmlinux/<kernel-version>/vmlinux` 파일이 있는지 확인합니다.
3. 파일이 있으면 기본 모드에서는 `crash <vmlinux> <vmcore>`를 실행합니다.
4. `-report` 모드에서는 `crash` 셸에 진입하지 않고 `/opt/kdump-tools/logs/kdump-report-<timestamp>.txt` 파일을 생성합니다.
5. LLM 분석이 활성화되어 있으면 생성된 리포트 내용을 API로 보내 원인분석, 근거, 영향 범위, 권고 조치를 리포트 끝에 추가합니다.
6. 파일이 없으면 필요한 `vmlinux` 준비 명령을 안내하고 종료합니다.

예상 출력 예시는 다음과 같습니다.

```text
vmcore:  /var/crash/127.0.0.1-2026-07-01-10:00:00/vmcore
kernel:  4.18.0-553.el8_10.x86_64
vmlinux: /opt/kdump-tools/vmlinux/4.18.0-553.el8_10.x86_64/vmlinux
```

리포트 모드에서는 다음과 같이 생성된 파일 경로를 출력합니다.

```text
report:  /opt/kdump-tools/logs/kdump-report-20260701-100000.txt
```

리포트에는 기본 메타데이터와 함께 `sys`, `bt`, `bt -a`, `ps`, `log`, `kmem -i`, `mount`, `files` 명령 결과가 포함됩니다.

LLM 분석이 활성화된 경우 리포트 끝에 `llm root cause analysis` 섹션이 추가됩니다. API 호출이나 응답 파싱에 실패해도 원본 crash 리포트는 유지되며, 실패 이유만 해당 섹션에 기록됩니다.

## 설정 파일

`analyze-kdump.conf.sample`은 bash 변수 형식으로 작성된 샘플입니다. 실제 운영 설정은 `/opt/kdump-tools/analyze-kdump.conf` 또는 `-config`로 지정한 파일에 작성합니다.

```bash
VMLINUX_BASE="/opt/kdump-tools/vmlinux"
REPORT_DIR="/opt/kdump-tools/logs"
REPORT_PREFIX="kdump-report"

# 선택 사항
DEFAULT_VMCORE="/var/crash/127.0.0.1-2026-07-01-10:00:00/vmcore"

# 선택 사항: LLM 분석
LLM_ENABLED=0
LLM_PROVIDER="openai"
LLM_MODEL="gpt-4o-mini"
LLM_MAX_TOKENS=2048
LLM_MAX_REPORT_CHARS=60000
LLM_TIMEOUT=120
```

설정 가능한 주요 항목은 다음과 같습니다.

- `VMLINUX_BASE`: 커널 버전별 `vmlinux` 파일을 찾을 기준 디렉터리
- `REPORT_DIR`: `-report` 결과 파일을 저장할 디렉터리
- `REPORT_PREFIX`: 생성되는 리포트 파일명 접두어
- `DEFAULT_VMCORE`: 명령줄 인자가 없을 때 사용할 기본 `vmcore` 경로
- `REPORT_COMMANDS`: 리포트에 포함할 `crash` 명령 목록
- `REPORT_COMMAND_FILE`: 별도 파일로 관리하는 `crash` 명령 목록
- `LLM_ENABLED`: `1`, `true`, `yes`, `on`으로 설정하면 `-report` 실행 시 LLM 분석 추가
- `LLM_API_KEY`: LLM API 키. conf 파일에 저장하기보다 환경변수로 주입하는 방식을 권장
- `LLM_API_KEY_FILE`: API 키를 담은 파일 경로. 파일 권한은 `600` 권장
- `LLM_PROVIDER`: `openai`, `gemini`, `claude` 중 하나. `openai`는 OpenAI 호환 Chat Completions API도 포함
- `LLM_BASE_URL`: provider 기본 엔드포인트 대신 사용할 API 엔드포인트
- `LLM_MODEL`: 사용할 모델명
- `LLM_MAX_TOKENS`: LLM이 생성할 최대 토큰 수
- `LLM_MAX_REPORT_CHARS`: LLM 입력으로 전달할 리포트 최대 문자 수
- `LLM_TIMEOUT`: LLM API 호출 제한 시간(초)
- `LLM_PROMPT_FILE`: 기본 원인분석 프롬프트 대신 사용할 프롬프트 파일
- `LLM_ANTHROPIC_VERSION`: Claude API에 전달할 Anthropic API 버전. 기본값은 `2023-06-01`

환경변수로도 기본값을 바꿀 수 있지만, 운영 환경에서는 conf 파일로 관리하는 방식을 권장합니다.

API 키는 다음처럼 실행 시 환경변수로 전달할 수 있습니다.

```bash
export LLM_API_KEY="<api-key>"
analyze-kdump -report -llm /path/to/vmcore
```

API 키를 파일로 저장해야 한다면 `secrets/` 디렉터리에 두고 권한을 제한합니다. 저장소 루트의 `secrets/`는 Git에 올라가지 않도록 `.gitignore`에 포함되어 있습니다.

```bash
mkdir -p /opt/kdump-tools/secrets
printf '%s\n' '<gemini-api-key>' > /opt/kdump-tools/secrets/gemini-api-key
chmod 600 /opt/kdump-tools/secrets/gemini-api-key
```

이 경우 실제 `analyze-kdump.conf`에는 다음처럼 지정합니다.

```bash
LLM_PROVIDER="gemini"
LLM_MODEL="gemini-3.5-flash"
LLM_API_KEY_FILE="/opt/kdump-tools/secrets/gemini-api-key"
```

Provider별 최소 설정 예시는 다음과 같습니다.

```bash
# OpenAI 또는 OpenAI 호환 API
LLM_PROVIDER="openai"
LLM_MODEL="gpt-4o-mini"

# Gemini
LLM_PROVIDER="gemini"
LLM_MODEL="gemini-3.5-flash"

# Claude
LLM_PROVIDER="claude"
LLM_MODEL="claude-sonnet-4-5"
```

## vmlinux 추가 방법

분석하려는 `vmcore`의 커널 버전에 해당하는 `vmlinux`가 없다면 디버그 심볼을 설치한 뒤 `/opt/kdump-tools/vmlinux` 아래에 복사합니다.

```bash
KVER="<kernel-version>"

dnf debuginfo-install -y "kernel-$KVER"
mkdir -p "/opt/kdump-tools/vmlinux/$KVER"
cp "/usr/lib/debug/lib/modules/$KVER/vmlinux" "/opt/kdump-tools/vmlinux/$KVER/"
```

예를 들어 커널 버전이 `4.18.0-553.el8_10.x86_64`라면 최종 파일 경로는 다음과 같아야 합니다.

```text
/opt/kdump-tools/vmlinux/4.18.0-553.el8_10.x86_64/vmlinux
```

## 로그 디렉터리

`logs/`는 분석 결과나 작업 기록을 남기기 위한 디렉터리입니다. `-report` 옵션으로 생성되는 요약 리포트가 이 디렉터리에 저장됩니다.
