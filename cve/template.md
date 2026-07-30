---
cve: CVE-YYYY-NNNNN
cwe: CWE-XXX
vulnerability: (OOB Write / UAF / OOM / Integer Overflow / ...)
target: (libtiff / libpng / libjpeg-turbo)
author: 이름
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
