# 파일 구조 / 라이브러리 구조 비교

분석자: 오동규

## 파일 구조 비교

|  | TIFF | PNG | JPEG |
| --- | --- | --- | --- |
| 구조 방식 | 오프셋 기반 (자유 배치) | chunk 순차 | marker segment 순차 |
| 바이트 오더 | 파일마다 다름 (II/MM) | 빅 엔디안 고정 | 빅 엔디안 고정 |
| 파일 식별 | 4바이트 (byte order + magic 42) | 8바이트 시그니처 | FF D8 (+ FF) |
| 메타데이터 | IFD/Tag (ID+Type+Count+Value) | chunk (Length+Type+Data+CRC) | marker segment (FF+Type+Length+Data) |
| 이미지 데이터 | strip/tile (개별 접근 가능) | IDAT (전체 이어붙여서 디코딩) | SOS 이후 연속 데이터 |
| 멀티 이미지 | IFD linked list로 지원 | 단일 이미지 | 단일 이미지 |
| 압축 | 다중 코덱 (LZW, JPEG, Deflate 등) | deflate 고정 | DCT+양자화+허프만 고정 |
| 손실/무손실 | 무손실 기본 (JPEG 코덱 사용 시 손실) | 무손실 | 손실 |
| 무결성 검증 | 없음 | CRC (chunk마다) | 없음 |
| 이미지 분할 | strip/tile | 없음 | 없음 |
| 최대 파일 크기 | 4GB (BigTIFF는 사실상 무제한) | 제한 없음 | 제한 없음 |

## 라이브러리 구조 비교

|  | libtiff | libpng | libjpeg-turbo |
| --- | --- | --- | --- |
| 소스 위치 | libtiff/ | 루트 | src/ |
| 메인 구조체 | TIFF (하나에 전부) | png_struct + png_info (분리) | jpeg_decompress/compress_struct (압축/해제 분리) |
| 이미지 정보 저장 | TIFFDirectory (tif_dir) | png_info | 구조체 내 필드 직접 |
| 코덱 처리 | 내부 코덱 파일 (tif_lzw.c 등) | zlib 외부 위임 | 내부 구현 (jdhuff.c, jidctint.c 등) |
| 정보 접근 | TIFFGetField (함수) | png_get_* (함수) | 구조체 필드 직접 접근 |
| 모듈화 | 코덱 함수 포인터 | zlib에 위임 | 서브모듈 포인터 (entropy, idct 등) |
| I/O | 콜백 함수 포인터 | 콜백 함수 포인터 | jpeg_source_mgr 구조체 |

## 공통점

- 세 라이브러리 모두 읽기(해제)와 쓰기(압축)를 지원
- 세 라이브러리 모두 I/O를 콜백/인터페이스로 추상화해서 파일, 메모리, 네트워크 등 다양한 소스에서 동작 가능
- API 흐름이 유사: 초기화 → 메타데이터 읽기 → 행 단위 데이터 읽기 → 정리

## 보안 관점에서의 특징

### TIFF

세 이미지 형식 중 가장 공격 표면이 넓다.

- 오프셋 기반 구조라서 파일 내 임의 위치를 가리킬 수 있다.
- tag의 value/offset 이중 해석이 가능하다.
- 다중 코덱을 지원하여 각 코덱마다 취약점이 나올 수 있다.
- IFD linked list로 순환 참조 문제가 발생 가능

### PNG

구조가 단순하고 제약이 많아서 상대적으로 공격 표면이 좁다.

- deflate 해제 과정(zlib), 필터 역적용, chunk 파싱에서 취약점이 발생할 수 있다.
- 공격자가 CRC를 의도적으로 조작할 수 있으므로 보안적 효과는 없다.

### JPEG

디코딩 파이프라인이 복잡하고 각 단계마다 취약점이 나올 수 있다(허프만 디코딩 → DCT 계수 → IDCT → 색상 변환 → 업샘플링).

- marker 파싱에서 취약점이 발생할 수 있다.
- byte stuffing 처리에서 취약점이 발생할 수 있다.
