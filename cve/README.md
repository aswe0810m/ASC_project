# CVE 분석

이미지 처리 라이브러리(libtiff, libpng, libjpeg-turbo)의 CVE를 분석한 문서 모음.

## 분석 목록

<!-- CVE_TABLE_START -->
| CVE | CWE | 취약점 | 대상 | 분석자 |
|-----|-----|--------|------|--------|
| [CVE-2019-7317](CVE-2019-7317.md) | CWE-416 | UAF | libpng | 오동규 |
| [CVE-2022-34526](CVE-2022-34526.md) | CWE-787 | OOB Write | libtiff | 오동규 |
| [CVE-2023-52355](CVE-2023-52355.md) | CWE-400 | OOM | libtiff | 안민기 |
| [CVE-2023-52356](CVE-2023-52356.md) |  |  | libtiff | 유창하 |
| [CVE-2023-6277](CVE-2023-6277.md) | CWE-400 | OOM | libtiff | 이화진 |
| [CVE-2025-9900](CVE-2025-9900.md) | CWE-123 | OOB Write | libtiff | 이화진 |

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
---
```

frontmatter가 있으면 분석 목록 테이블이 자동으로 갱신된다.
