# PNG 및 libpng 분석

분석자: 오동규

# PNG 파일 포맷 분석

## PNG 전체 파일 구조

---

1. Signature (8 bytes)
2. Chunk 구조
3. 필수 chunk
4. 보조 chunk

## Signature

---

PNG 파일은 항상 다음 8 바이트로 시작한다.

`89 50 4E 47 0D 0A 1A 0A` 

PNG는 항상 바이트 오더가 빅 엔디안으로 고정이기 때문에 TIFF처럼 바이트 오더를 명시할 필요가 없다.

## Chunk 구조

---

시그니처 다음에 **chunk**들이 연속적으로 이어지는 구조를 가진다. PNG의 모든 데이터는 **chunk** 단위로 저장된다. 각 chunk의 구조:

- **Length (4 bytes)** - 해당 chunk의 data 부분 크기
- **Type (4 bytes)** - chunk의 종류를 나타내는 4글자 ASCII (예: `IHDR`, `IDAT`, `IEND`)
- **Data (Length bytes)** - 실제 데이터
- **CRC (4 bytes)** - Type + Data에 대한 체크섬 (무결성 검증)

chunk를 다 읽으면 바로 다음 chunk가 시작되는 순차적 구조이다.

## 필수 chunk

---

PNG 파일에는 반드시 `IHDR`, `IDAT`, `IEND` chunk가 존재하여야 한다.

### IHDR (Image Header)

반드시 첫 번째 chunk로 이미지의 기본 정보를 담고 있다.

- **Width (4 bytes)** - 너비
- **Height (4 bytes)** - 높이
- **Bit depth (1 byte)** - 색상의 bit 수
- **Color type (1 byte)** - 색상 종류 (`grayscale`, `RGB`, `palette` 등)
- **Compression method (1 byte)** - 압축 방식 deflate (0)으로 고정되어 있다.
- **Filter method (1 byte)** - 압축 전 픽셀 데이터를 변환하여 압축 효율을 높이는 필터링 방식 adaptive filtering (0)으로 고정되어 있다.
- **Interlace method (1 byte)** - 이미지 순서 배치 방법 (현재는 의미있는 정보는 아니다)

### IDAT (Image Data)

실제 압축된 픽셀 데이터로 여러 개의 IDAT chunk로 나뉠 수 있다.

**압축 과정**

원본 픽셀 → 각 행 앞에 필터 바이트 추가 → 필터 적용 → deflate 압축 → IDAT에 저장

디코딩은 이 역순으로 적용된다.

이때, 압축된 데이터가 크면 하나의 IDAT에 다 넣지 않고 여러 IDAT chunk로 쪼개서 저장할 수 있다. 쪼개서 저장한다고 TIFF의 split/tile같은 개념은 아니다. 쪼개진 데이터를 각각 디코딩 할 수 있는 것이 아닌, 전체를 이어붙인 후 deflate 해제를 하여야지 전체 이미지를 얻을 수 있기 때문이다.

IDAT 청크들은 연속적으로 존재하여야 한다. 중간에 다른 tEXt같은 청크가 들어가면 안 된다.

### IEND (Image End)

파일의 끝을 표시하는 청크로 Data 크기가 0인 빈 chunk이다.

## 보조 chunk

---

필수는 아니지만 자주 쓰이는 chunk들:

- **PLTE** - 팔레트 색상 테이블 (palette 기반 이미지에서 필수)
- **tEXt** - 텍스트 메타데이터
- **gAMA** - 감마 값
- **tRNS** - 투명도 정보
- **iCCP** - ICC 색상 프로파일

## PNG 특징

---

- 순차 구조로 설계되어 있다.
- 바이트 오더가 항상 빅 엔디안으로 고정되어 있다.
- chunk 단위로 데이터가 저장된다. chunk마다 CRC로 무결성 검증을 한다.
- 압축 방식이 deflate로 고정되어 있다.
- 이미지 분할이 없고 단일 이미지를 저장한다.

# libpng 분석

## 기본 파일 구조

---

서브 디렉토리 없이 `.c`, `.h` 파일들이 병렬적으로 나열된 구조이다.

### 헤더 파일

- `png.h` - 외부 API 선언
- `pngpriv.h` - 내부 함수/매크로 선언
- `pngstruct.h` - `png_struct` 구조체 정의
- `pnginfo.h` - `png_info` 구조체 정의
- `pngconf.h` - 컴파일러/플랫폼 설정

### 읽기

- `pngread.c` - 읽기 흐름 제어 (초기화, chunk 읽기 순서 관리)
- `pngrutil.c` - chunk 파싱 (IHDR, IDAT, PLTE 등 실제 처리)
- `pngrtran.c` - 읽기 시 데이터 변환 (감마, 색상 변환)
- `pngrio.c` - 읽기 I/O

### 쓰기

- `pngwrite.c` - 쓰기 흐름 제어
- `pngwutil.c` - chunk 쓰기
- `pngwtran.c` - 쓰기 시 데이터 변환
- `pngwio.c` - 쓰기 I/O

### 공통

- `png.c` - 공통 함수, 라이브러리 초기화
- `pngtrans.c` - 공통 변환 (필터 등)
- `pngget.c` - 이미지 정보 getter
- `pngset.c` - 이미지 정보 setter
- `pngmem.c` - 메모리 할당/해제
- `pngerror.c` - 에러/경고 처리
- `pngpread.c` - progressive(점진적) 읽기

### 기타 디렉토리

- `arm/`, `intel/`, `mips/` 등 - SIMD 최적화 코드
- `contrib/` - 보조 도구
- `tests/` - 테스트 도구

## 구조체 정리

---

libpng는 핵심 구조체가 두 개이다.

- `png_struct` - 라이브러리의 동작 상태를 담는 구조체. 압축/해제 상태, I/O 콜백, 에러 핸들러, 현재 처리 중인 행 번호 등. `pngstruct.h` 에 정의되어 있다.
- `png_info` - 이미지의 메타데이터를 담는 구조체. width, height, bit depth, color type, 압축 방식 등. `pnginfo.h` 에 정의되어 있다.

## png_struct

라이브러리 동작 상태.

### 에러/콜백

- `error_fn` / `warning_fn` - 에러/경고 핸들러
- `read_data_fn` / `write_data_fn` - I/O 콜백
- `io_ptr` - I/O 콜백에 넘겨지는 사용자 데이터
- `jmp_buf_local` / `longjmp_fn` - 에러 발생시 비로컬 점프 (libpng의 에러 처리 방식)

### 상태/플래그

- `mode` - 현재 파일 처리 단계 (IHDR 읽는 중, IDAT 읽는 중 등)
- `flags` - 각종 상태 플래그
- `transformations` - 적용할 변환 목록 (감마, 색상 변환 등)

### zlib (압축/해제)

- `zstream` - zlib의 압축/해제 상태 구조체. libpng가 직접 압축을 구현하지 않고 zlib에 위임하는 부분.
- `zlib_level`, `zlib_method` 등 - zlib 압축 설정값들

### 이미지 기본 정보 (IHDR에서 파싱)

`png_info` 에 저장되어 있지만 빠르게 접근하기 위한 캐시로 사용된다.

- `width`, `height` - 이미지 크기
- `bit_depth`, `color_type` - 비트 깊이, 색상 타입
- `compression`, `filter`, `interlaced` - 압축/필터/인터레이스 방식
- `channels` - 채널 수

### 행 버퍼

- `row_buf` - 현재 행 데이터 버퍼 (필터 해제된 상태)
- `prev_buf` - 이전 행 데이터 (Up, Average, Paeth 필터 역적용에 필요)
- `row_number` - 현재 처리 중인 행 번호

### CRC/chunk

- `crc` - 현재 chunk의 CRC 값
- `chunk_name` - 현재 처리 중인 chunk 이름
- `idat_size` - 현재 IDAT chunk의 크기

## png_info

이미지 메타데이터

### 필수 정보 (IHDR)

- `width`, `height` - 이미지 크기
- `bit_depth`, `color_type` - 비트 깊이, 색상 타입
- `compression_type`, `filter_type`, `interlace_type` - 압축/필터/인레이스
- `channels`, `pixel_depth` - 채널 수, 픽셀 당 비트 수
- `rowbytes` - 한 행의 바이트 크기
- `valid` - 어떤 chunk 정보가 유효한지 비트 플래그

### 보조 chunk 정보

- `palette`, `num_palette` - PLTE chunk (팔레트 색상)
- `trans_alpha`, `trans_color` - tRNS chunk (투명도)
- `background` - bKGD chunk (배경색)
- `text`, `num_text` - tEXt chunk (텍스트 메타데이터)
- `gamma` - gAMA chunk
- `x_pixels_per_unit`, `y_pixels_per_unit` - pHYs chunk (해상도)

## API 함수 정리

---

### 초기화/정리

- `png_create_read_struct` - 읽기용 `png_struct` 생성
- `png_create_write_struct` - 쓰기용 `png_struct` 생성
- `png_create_info_struct` - `png_info` 생성
- `png_destroy_read_struct` - 읽기 관련 구조체 해제
- `png_destroy_write_struct` - 쓰기 관련 구조체 해제

### I/O 설정

- `png_init_io` - 파일 포인터(`FILE *`) 연결
- `png_set_read_fn` - 커스텀 읽기 콜백 설정
- `png_set_write_fn` - 커스텀 쓰기 콜백 설정

### 읽기

- `png_read_info` - IHDR~IDAT 사이의 chunk들을 읽어서 `png_info` 에 저장
- `png_read_row` - 한 행 읽기
- `png_read_rows` - 여러 행 읽기
- `png_read_image` - 이미지 전체를 한 번에 읽기
- `png_read_end` - IDAT 이후 chunk들 읽기 (tEXt 등)

### 쓰기

- `png_write_info` - 메타데이터 chunk들 쓰기
- `png_write_row` - 한 행 쓰기
- `png_write_image` - 이미지 전체 쓰기
- `png_write_end` - 마무리 chunk (IEND 등) 쓰기

### 정보 접근 (get/set)

- `png_get_IHDR` / `png_set_IHDR` - IHDR 정보 한번에 가져오기/설정
- `png_get_image_width` - 이미지 너비
- `png_get_image_height` - 이미지 높이
- `png_get_bit_depth` - 비트 깊이
- `png_get_color_type` - 색상 타입
- `png_get_PLTE` / `png_set_PLTE` - 팔레트

### 변환 설정

- `png_set_expand` - 1/2/4 비트를 8비트로 확장
- `png_set_strip_16` - 16비트를 8비트로 축소
- `png_set_gray_to_rgb` - 그레이스케일을 RGB로 변환
- `png_set_add_alpha` - 알파 채널 추가
- `png_set_gamma` - 감마 보정

# 참고 문헌

## PNG

---

### PNG 파일 포맷

- PNG 공식 스펙 (W3C): [https://www.w3.org/TR/png/](https://www.w3.org/TR/png/) — Signature, chunk 구조, IHDR 필드, IDAT, 필터링, CRC 등

### libpng

- 공식 저장소: [https://github.com/pnggroup/libpng](https://github.com/pnggroup/libpng)
- libpng 매뉴얼: [http://www.libpng.org/pub/png/libpng-manual.txt](http://www.libpng.org/pub/png/libpng-manual.txt) — API 함수, `png_struct`/`png_info` 설명