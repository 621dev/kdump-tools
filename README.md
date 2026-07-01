# kdump-tools

`kdump-tools`는 Linux `vmcore` 분석을 조금 더 빠르게 시작하기 위한 보조 도구 모음입니다.

현재는 `vmcore`에서 커널 버전을 확인한 뒤, 해당 버전에 맞는 `vmlinux` 디버그 심볼을 찾아 `crash`를 실행하는 `analyze-kdump` 스크립트를 제공합니다.

## 주요 기능

- `vmcore` 파일의 커널 버전 자동 확인
- 커널 버전에 맞는 `vmlinux` 경로 자동 매칭
- 매칭되는 `vmlinux`가 없을 때 필요한 설치 명령 안내
- `crash <vmlinux> <vmcore>` 실행 자동화
- `-report` 옵션으로 셸 진입 없이 요약 리포트 파일 생성

## 디렉터리 구조

권장 설치 위치는 `/opt/kdump-tools`입니다.

```text
/opt/kdump-tools/
├── README.md
├── analyze-kdump.conf
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

요약 리포트만 생성하려면 `-report` 옵션을 사용합니다.

```bash
/opt/kdump-tools/bin/analyze-kdump -report /path/to/vmcore
```

기본 설정 파일은 `/opt/kdump-tools/analyze-kdump.conf`입니다. 다른 설정 파일을 사용하려면 `-config` 옵션으로 지정합니다.

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
5. 파일이 없으면 필요한 `vmlinux` 준비 명령을 안내하고 종료합니다.

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

## 설정 파일

`analyze-kdump.conf`는 bash 변수 형식으로 작성합니다.

```bash
VMLINUX_BASE="/opt/kdump-tools/vmlinux"
REPORT_DIR="/opt/kdump-tools/logs"
REPORT_PREFIX="kdump-report"

# 선택 사항
DEFAULT_VMCORE="/var/crash/127.0.0.1-2026-07-01-10:00:00/vmcore"
```

설정 가능한 주요 항목은 다음과 같습니다.

- `VMLINUX_BASE`: 커널 버전별 `vmlinux` 파일을 찾을 기준 디렉터리
- `REPORT_DIR`: `-report` 결과 파일을 저장할 디렉터리
- `REPORT_PREFIX`: 생성되는 리포트 파일명 접두어
- `DEFAULT_VMCORE`: 명령줄 인자가 없을 때 사용할 기본 `vmcore` 경로
- `REPORT_COMMANDS`: 리포트에 포함할 `crash` 명령 목록
- `REPORT_COMMAND_FILE`: 별도 파일로 관리하는 `crash` 명령 목록

환경변수로도 기본값을 바꿀 수 있지만, 운영 환경에서는 conf 파일로 관리하는 방식을 권장합니다.

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
