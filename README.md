# ASC 1조 팀 프로젝트 - 이미지 라이브러리 CVE 분석

이미지 처리 라이브러리(libtiff, libpng, libjpeg-turbo)의 CVE를 체계적으로 분석하는 프로젝트.

## 프로젝트 구조

```
├── library-analysis/       # 라이브러리 선행 분석
│   ├── README.md           # 파일 구조 / 라이브러리 구조 비교표
│   ├── libtiff.md          # TIFF 파일 포맷 + libtiff 분석
│   ├── libpng.md           # PNG 파일 포맷 + libpng 분석
│   └── libjpeg-turbo.md    # JPEG 파일 포맷 + libjpeg-turbo 분석
├── cve/                    # CVE 개별 분석
│   ├── README.md           # CVE 분석 목록 + 기여 가이드
│   ├── template.md         # 분석 템플릿
│   └── images/
└── references/             # 참고 자료
    ├── hwajin-report.md    # 라이브러리 분석 보고서
    └── function-reference.md # 핵심 함수 비교 레퍼런스
```

## CVE 분석 목록

<!-- CVE_TABLE_START -->
| CVE | CWE | 취약점 | 대상 | 분석자 |
|-----|-----|--------|------|--------|
| [CVE-2018-19664](cve/CVE-2018-19664.md) | CWE-125 | OOB Read | libjpeg-turbo | 안민기 |
| [CVE-2019-7317](cve/CVE-2019-7317.md) | CWE-416 | UAF | libpng | 오동규 |
| [CVE-2022-34526](cve/CVE-2022-34526.md) | CWE-787 | OOB Write | libtiff | 오동규 |
| [CVE-2023-25433](cve/CVE-2023-25433.md) | CWE-120 | Heap Buffer Overflow | libtiff | 안민기 |
| [CVE-2023-52355](cve/CVE-2023-52355.md) | CWE-400 | OOM | libtiff | 안민기 |
| [CVE-2023-52356](cve/CVE-2023-52356.md) |  |  | libtiff | 유창하 |
| [CVE-2023-6277](cve/CVE-2023-6277.md) | CWE-400 | OOM | libtiff | 이화진 |
| [CVE-2025-9900](cve/CVE-2025-9900.md) | CWE-123 | OOB Write | libtiff | 이화진 |

<!-- CVE_TABLE_END -->

## 라이브러리 선행 분석

| 라이브러리 | 분석 문서 |
|-----------|----------|
| libtiff | [TIFF 및 libtiff 분석](library-analysis/libtiff.md) |
| libpng | [PNG 및 libpng 분석](library-analysis/libpng.md) |
| libjpeg-turbo | [JPEG 및 libjpeg-turbo 분석](library-analysis/libjpeg-turbo.md) |
| 비교 개요 | [파일 구조 / 라이브러리 구조 / 보안 특징 비교](library-analysis/) |
