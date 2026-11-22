# CVE 분석

이미지 처리 라이브러리(libtiff, libpng, libjpeg-turbo)의 CVE를 분석한 문서 모음.

## 분석 목록

<!-- CVE_TABLE_START -->
| CVE | CWE | 취약점 | 대상 | 분석자 |
|-----|-----|--------|------|--------|
| [CVE-2018-19664](CVE-2018-19664.md) | CWE-125 | OOB Read | libjpeg-turbo | 안민기 |
| [CVE-2019-7317](CVE-2019-7317.md) | CWE-416 | UAF | libpng | 오동규 |
| [CVE-2020-13790](CVE-2020-13790.md) | CWE-125 | OOB Read | libjpeg-turbo | Hwajin |
| [CVE-2020-17541](CVE-2020-17541.md) | CWE-787 | OOB Write | libjpeg-turbo | 오동규 |
| [CVE-2022-34526](CVE-2022-34526.md) | CWE-787 | OOB Write | libtiff | 오동규 |
| [CVE-2023-25433](CVE-2023-25433.md) | CWE-120 | Heap Buffer Overflow | libtiff | 안민기 |
| [CVE-2023-2804](CVE-2023-2804.md) | CWE-125 | OOB Read | libjpeg-turbo | 오동규 |
| [CVE-2023-52355](CVE-2023-52355.md) | CWE-400 | OOM | libtiff | 안민기 |
| [CVE-2023-52356](CVE-2023-52356.md) |  |  | libtiff | 유창하 |
| [CVE-2023-6277](CVE-2023-6277.md) | CWE-400 | OOM | libtiff | 이화진 |
| [CVE-2025-64505](CVE-2025-64505.md) | CWE-125 | OOB Read | libpng | 오동규 |
| [CVE-2025-65018](CVE-2025-65018.md) | CWE-122 | Heap Buffer Overflow | libpng | 오동규 |
| [CVE-2025-9900](CVE-2025-9900.md) | CWE-123 | OOB Write | libtiff | 이화진 |
| [CVE-2026-25646](CVE-2026-25646.md) | CWE-126 | OOB Read | libpng | 오동규 |

<!-- CVE_TABLE_END -->

## 분석 형식

각 CVE 분석은 다음 순서로 작성한다. 템플릿은 [template.md](template.md)를 참고. 필요에 따라 내용을 가감하여 작성한다.

1. **NVD** — Description, CVSS 점수, NVD에서 알 수 있는 사실 정리
2. **git diff** — 패치 커밋의 diff를 분석하여 NVD 내용 구체화
3. **취약점 원인 분석** — 구조체 분석, 취약 코드 경로, 핵심 원인 정리
4. **PoC 분석** — PoC 코드 분석, 재현 환경/절차, 출력 결과

## 파일 추가 방법

1. 브랜치를 생성한다 (예: `CVE-2024-XXXXX`)
2. `template.md`를 복사하여 `CVE-2024-XXXXX.md`로 작성한다
3. 상단 frontmatter(YAML)를 반드시 채운다
4. 이미지가 필요하면 `images/`에 추가하고 마크다운에서 참조한다
5. PR을 생성한다

### frontmatter 형식

```yaml
---
cve: CVE-2024-XXXXX
cwe: CWE-XXX
vulnerability: OOB Write
target: libtiff
author: 이름
# ===== 구조화 필드 =====
stage: 메타데이터 파싱
weakness_class: 메모리 경계 위반
vuln_function: 함수명
vuln_file: 파일 경로
entry_point: 외부 API 또는 도구
trigger_path: "entry_point → ... → vuln_function"
input_manipulation: "입력 파일의 어떤 태그/청크/마커를 어떻게 조작하는지"
build_condition:
root_cause_pattern: "근본 원인 한 줄"
patch_type: 검증 추가
poc: reproduced
verified_by:
---
```

`cve`, `cwe`, `vulnerability`, `target`, `author`가 있으면 분석 목록 테이블이 자동으로 갱신된다. 값은 한 줄로 쓰고, 콜론(`:`)이나 `#`이 들어가면 큰따옴표로 감싼다. 모르는 값은 비워둔다.

### 구조화 필드

| 필드 | 내용 |
|------|------|
| `stage` | 어디서: 취약 코드가 속한 처리 단계 (아래 분류 기준) |
| `weakness_class` | 어떤 취약점: 실제로 일어난 결과의 묶음 (아래 분류 기준) |
| `vuln_function` / `vuln_file` | 취약점이 실제로 발생하는 함수와 파일 |
| `entry_point` | 외부에서 들어오는 API 또는 도구 |
| `trigger_path` | entry_point에서 vuln_function까지의 호출 흐름 |
| `input_manipulation` | 입력 파일의 어떤 부분을 어떻게 조작하는지 |
| `build_condition` | 재현에 필요한 빌드 조건 (예: `--disable-zlib`), 없으면 비움 |
| `root_cause_pattern` | 근본 원인 한 줄 (자유 서술) |
| `patch_type` | 검증 추가 / 크기 계산 수정 / 버퍼 확장 / 로직 제거 / 기타 |
| `poc` | none / public / reproduced (`reproduced`는 직접 재현했을 때만) |
| `verified_by` | diff와 재현으로 내용을 검증한 사람. AI 초안이면 검증 전까지 비움 |

### 분류 기준

**stage (어디서)**: `Init` / `메타데이터 파싱` / `디코딩` / `인코딩` / `기타`

- 취약 코드가 속한 처리 단계로 정한다.
- 라이브러리 핵심 파이프라인이 아닌 경우(도구 로직, 보조 포맷 입출력 등)는 `기타`로 둔다.

**weakness_class (어떤 취약점)**: 실제로 일어난 **결과**로 분류한다.

| weakness_class | 해당하는 결과 (sanitizer 출력 등) |
|------|------|
| 메모리 경계 위반 | heap/stack/global-buffer-overflow (READ/WRITE) |
| 수명 관리 오류 | heap-use-after-free, stack-use-after-return/scope, double-free |
| 자원 고갈 | 거대 할당(allocation-size-too-big), OOM, 메모리 누수, 무한 루프/timeout |
| 기타 | NULL 역참조, 0으로 나누기, 미초기화 메모리 사용, assertion 실패 등 |

- 근거 우선순위: 재현 결과(sanitizer 출력) > 코드 분석 > NVD의 CWE(참고만).
- 정수 오버플로우처럼 원인이 따로 있으면 결과로 분류하고, 원인은 `root_cause_pattern`에 적는다.
- 결과가 여러 개면 재현에서 처음 관찰된 것으로 분류한다.
- 어디에도 안 맞으면 `기타`에 넣고, 비슷한 것이 2~3개 쌓이면 새 묶음으로 승격한다. 새 묶음 추가는 PR에서 오너가 승인한다.
- 원본 `cwe`는 항상 함께 적는다 (나중에 재분류할 때 필요).
