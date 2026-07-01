# kdump-tools

kdump 분석 작업을 편리하게 진행할 수 있도록 돕는 유틸리티 및 리소스 관리 디렉터리입니다. `vmcore` 파일의 커널 버전을 자동으로 판별하여 적절한 `vmlinux` 디버그 심볼 파일을 매칭하고 `crash` 분석 도구를 실행해 줍니다.

---

## 📂 디렉터리 구조

```text
/opt/kdump-tools/
├── README.md             # 본 문서
├── bin/                  # 분석 실행 스크립트 디렉터리
│   └── analyze-kdump     # vmcore 분석 자동화 스크립트
├── logs/                 # 로그 보관용 디렉터리
├── packages/             # 커널 디버그인포 관련 RPM 패키지 보관 디렉터리
└── vmlinux/              # 커널 버전별 디버그 심볼(vmlinux) 보관 디렉터리
    ├── 4.18.0-553.el8_10.x86_64/
    │   └── vmlinux
    └── 4.18.0-553.137.1.el8_10.x86_64/
        └── vmlinux
```

---

## 🛠️ 주요 구성 요소 설명

### 1. `bin/analyze-kdump`
* **역할**: `vmcore` 파일의 커널 버전을 자동으로 추출하여 이에 대응하는 `vmlinux` 파일로 `crash` 세션을 시작합니다.
* **사용법**:
  ```bash
  /opt/kdump-tools/bin/analyze-kdump <vmcore_파일_경로>
  ```
* **동작 원리**:
  1. `crash --osrelease <vmcore>` 명령을 통해 vmcore의 커널 버전을 확인합니다.
  2. `/opt/kdump-tools/vmlinux/<커널_버전>/vmlinux` 디렉터리에 매칭되는 디버그 심볼이 있는지 검사합니다.
  3. 일치하는 심볼 파일이 존재하면 `crash <vmlinux> <vmcore>`를 실행합니다.
  4. 없을 경우, 알맞은 버전의 `kernel-debuginfo` 설치 안내 메시지를 출력합니다.

### 2. `vmlinux/`
* `crash` 디버깅에 필수적인 커널 디버그 심볼 파일(`vmlinux`)들을 버전별 폴더에 보관합니다.
* **현재 보관 중인 버젼**:
  * `4.18.0-553.el8_10.x86_64`
  * `4.18.0-553.137.1.el8_10.x86_64`

### 3. `packages/`
* 오프라인 환경 등에서 커널 디버그 심볼을 수동 설치할 수 있도록 관련 RPM 패키지를 보관하는 용도입니다.
* **현재 보관 중인 패키지**:
  * `kernel-debuginfo-4.18.0-553.el8_10.x86_64.rpm`
  * `kernel-debuginfo-common-x86_64-4.18.0-553.el8_10.x86_64.rpm`

### 4. `logs/`
* 분석 과정이나 실행 결과 등을 남길 수 있는 로그용 빈 디렉터리입니다.

---

## 💡 새로운 커널 버전에 대한 vmlinux 추가 방법

분석하려는 `vmcore`의 커널 버전에 해당하는 `vmlinux`가 없는 경우, 다음과 같이 심볼을 추가해 주어야 합니다.

1. **디버그인포 패키지 다운로드 및 설치**
   ```bash
   dnf debuginfo-install -y kernel-<커널버전>
   ```
2. **vmlinux 보관 디렉터리 생성**
   ```bash
   mkdir -p /opt/kdump-tools/vmlinux/<커널버전>
   ```
3. **vmlinux 복사**
   ```bash
   cp /usr/lib/debug/lib/modules/<커널버전>/vmlinux /opt/kdump-tools/vmlinux/<커널버전>/
   ```
