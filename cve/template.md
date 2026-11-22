---
cve: CVE-YYYY-NNNNN
cwe: CWE-XXX
vulnerability: (OOB Write / UAF / OOM / Integer Overflow / ...)
target: (libtiff / libpng / libjpeg-turbo)
author: 이름
# ===== 구조화 필드 (DB / targeted fuzzing용) =====
# 값은 한 줄로 쓴다. 값에 콜론(:)이나 #이 들어가면 큰따옴표로 감싼다. 모르면 비워둔다.
# --- 분류 (어디서 + 어떤) ---
# stage: 취약 코드가 속한 처리 단계. 라이브러리 핵심 파이프라인이 아니면(도구 로직, 보조 포맷 입출력 등) 기타
# weakness_class: 실제로 일어난 결과로 분류. 근거 우선순위 = 재현(sanitizer 출력) > 코드 분석 > NVD CWE(참고)
#   메모리 경계 위반: heap/stack/global-buffer-overflow (READ/WRITE)
#   수명 관리 오류: use-after-free, stack-use-after-return/scope, double-free
#   자원 고갈: 거대 할당, OOM, 메모리 누수, 무한 루프/timeout
#   기타: NULL 역참조, 0으로 나누기, 미초기화 메모리, assertion 실패 등 (비슷한 게 2~3개 쌓이면 새 묶음으로 승격)
#   원인(정수 오버플로우 등)은 root_cause_pattern에, 결과가 여럿이면 처음 관찰된 것으로
stage:               # Init / 메타데이터 파싱 / 디코딩 / 인코딩 / 기타
weakness_class:      # 메모리 경계 위반 / 수명 관리 오류 / 자원 고갈 / 기타
vuln_function:       # 취약점이 실제로 발생하는 함수 (예: _TIFFVGetField)
vuln_file:           # 해당 함수가 있는 파일 (예: libtiff/tif_dir.c)
entry_point:         # 외부에서 들어오는 API/도구 (예: tiffcrop, png_image_begin_read_from_memory)
trigger_path:        # entry_point → ... → vuln_function 호출 흐름
input_manipulation:  # 입력 파일의 어떤 태그/청크/마커를 어떻게 조작하는지
build_condition:     # 재현에 필요한 빌드 조건 (예: --disable-zlib), 없으면 비움
root_cause_pattern:  # 근본 원인을 한 줄로 (자유 서술, 데이터가 쌓이면 분류 예정)
patch_type:          # 검증 추가 / 크기 계산 수정 / 버퍼 확장 / 로직 제거 / 기타
poc:                 # none / public / reproduced
verified_by:         # diff와 재현으로 내용을 검증한 사람 (AI 초안이면 검증 전까지 비움)
---

# CVE-YYYY-NNNNN (라이브러리 이름)

## NVD

### Description

> NVD의 Description을 여기에 인용한다.

### CVSS Version 3.x

- **Base Score:** X.X (Low / Medium / High / Critical)
- **Vector:** CVSS:3.1/AV:_/AC:_/PR:_/UI:_/S:_/C:_/I:_/A:_

### 알 수 있는 사실

- NVD Description과 CVSS에서 알 수 있는 사실들을 정리한다.
- 어떤 함수에서 문제가 발생하는지, 어떤 종류의 취약점인지 등.

## git diff

```bash
git clone <라이브러리 저장소 URL>
git log --oneline --all --grep="CVE-YYYY-NNNNN"  # 패치 커밋 해시 찾기
git diff <패치커밋>~1 <패치커밋>
```

패치 diff를 분석한다. 어떤 파일의 어떤 함수가 변경되었는지, 변경 전후의 코드를 비교한다.

```c
// diff 내용 중 중요 부분 삽입
```

NVD에서 파악한 내용이 diff를 통해 어떻게 구체화되는지 서술한다.

## 취약점 원인 분석

### 구조체 분석

취약점과 관련된 핵심 구조체를 분석한다.

```c
// 관련 구조체 정의
```

### 취약 코드 분석

취약점이 발생하는 코드 경로를 분석한다. 함수 호출 흐름, 변수의 상태 변화, 왜 취약점이 발생하는지를 단계별로 서술한다.

```c
// 취약 함수 코드
```

1. (단계별 흐름 설명)
2. ...

### 핵심 원인 정리

취약점의 근본 원인을 한 문단으로 요약한다.

## PoC 분석

PoC 출처 링크와 코드를 분석한다.

```c
// PoC 코드
```

PoC가 어떤 원리로 취약점을 트리거하는지 분석한다.

### 재현

재현 환경과 절차를 기록한다.

| 항목 | 값 |
|------|-----|
| OS | Ubuntu 22.04 (Docker) |
| Compiler | Clang / GCC |
| 대상 버전 | 라이브러리 vX.X.X |
| Sanitizer | ASan |

```bash
# 빌드 및 실행 명령어
```

### 출력 결과

```
# ASan 출력 등 재현 결과
```
