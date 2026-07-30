# TIFF 및 libtiff 분석

분석자: 오동규

# TIFF 파일 포맷 분석

## TIFF의 전체 구조

---

1. Header (8 bytes)
2. IFD (Image File Directory)
3. Tag entries
4. strip/tile

## Header

---

파일의 시작 지점으로 파일의 기본적인 정보들을 담고 있다.

- **Byte 0~1: Byte Order (리틀/빅 엔디안)**
- **Byte 2~3: Magic Number (42)**
- **Byte 4~7: IFD Offset**

### Byte Order

파일 전체에서 멀티 바이트 값을 읽는 순서를 결정한다.

- 리틀 엔디안: `II` (0x4949)
- 빅 엔디안: `MM` (0x4D4D)

I는 Intel을 의미하고, M은 Motorola를 의미한다.

### Magic Number

항상 42 (0x002A)의 값을 가진다. TIFF 파일임을 식별하는 값이다.

### IFD Offset

첫 번째 IFD가 파일 내에서 어디에 위치하는지를 가리키는 4바이트 오프셋이다. 파일 시작(byte 0)을 기준으로 한 절대 위치이다.

### Header 예시

`49 49 2A 00 08 00 00 00` : 리틀 엔디안 기반 tiff 파일, Header가 끝나자마자(8바이트) 첫 번째 IFD가 시작된다.

`4D 4D 00 2A 00 00 10 00` : 빅 엔디안 기반 tiff 파일, 4096(0x1000) 바이트에서 첫 번째 IFD가 시작된다.

## IFD (Image File Directory)

---

IFD는 하나의 이미지를 설명하는 메타 데이터 묶음이다. 구조는 다음과 같다:

- **Entry Count (2 bytes)**
- **Tag Entries (12 bytes * N)**
- **Next IFD offset (4 bytes)**

### Entry Count

해당 IFD에 tag entry가 몇 개 들어있는지를 나타내는 값이다. 파서는 이걸 읽고 그 뒤로 몇 개의 tag를 읽어야 하는지 결정한다.

### Tag Entries

실제 메타 데이터 항목들. 이미지의 너비, 높이, 압축 방식, 색상 모드같은 정보가 각각 하나의 Tag Entry로 저장된다. Tag Entry는 아래서 자세하게 다룬다.

### Next IFD offset

다음 IFD의 위치를 가리키는 오프셋이다. 0인 경우 해당 IFD가 마지막임을 의미한다.

이 구조를 통해서 TIFF는 단순한 이미지 형식이 아니라 이미지 묶음(컨테이너) 포맷에 가까운 것을 알 수 있다.

헤더의 IFD Offset → 1번 IFD → next offset → 2번 IFD → next offset → 3번 IFD → next offset = 0 (끝)

## Tag Entry

---

Tag Entry는 하나의 메타 데이터 항목으로, 각각 12바이트로 고정이다. 순서대로 다음 field를 갖는다.

1. Tag ID (2 bytes) - 이 항목이 어떤 정보인지를 나타내는 번호이다. 스펙에 정의되어 있어서, 번호만 보면 무엇을 하는지 알 수 있다. 자주 사용되는 값들은 다음과 같다:
    
    ```c
    - 256 (0x0100): ImageWidth (너비)
    - 257 (0x0101): ImageLength (높이)
    - 258 (0x0102): BitsPerSample (픽셀당 비트 수)
    - 259 (0x0103): Compression (압축 방식)
    - 273 (0x0111): StripOffsets (이미지 데이터 위치)
    - 279 (0x0117): StripByteCounts (각 strip의 크기)
    ```
    
2. Data Type (2 bytes) - 값의 타입을 나타낸다. 주요 타입은 다음과 같다:
    
    ```c
    - 1 = BYTE (1바이트)
    - 2 = ASCII (1바이트)
    - 3 = SHORT (2바이트)
    - 4 = LONG (4바이트)
    - 5 = RATIONAL (8바이트, 분자 4바이트 + 분모 4바이트)
    ```
    
3. Count (4 bytes) - 해당 타입의 값이 몇 개인지를 나타낸다.
    - 예를 들어 Data Type이 SHORT이고 Count가 3이면 2바이트 짜리 값이 3개로 총 6바이트이다.
4. Value/Offset (4 bytes) - 저장되는 데이터의 크기에 따라 두 가지 경우로 나뉜다.
    - 4 바이트 이하인 경우 → 이 필드에 값 자체가 들어간다.
    - 4 바이트 초과인 경우 → 실제 데이터가 저장된 위치의 오프셋이 들어간다.

예시)

`00 01 04 00 01 00 00 00 80 07 00 00` : ImageWidth가 1920임을 의미한다. LONG 1개 = 4바이트 이므로 Value로 들어간 경우이다.

`02 01 03 00 03 00 00 00 00 15 00 00` : BitsPerSample의 데이터가 0x00001500에 저장되어 있음을 의미한다. SHORT 3개 = 6바이트 이므로 Offset으로 들어간 경우이다. 8, 8, 8이 들어가 있다면 R, G, B 각각 8비트임을 의미한다.

이러한 Tag Entry들이 여러개 묶음으로 묶여 있는 것이 Tag Entries이다. 그리고 기본적으로 Baseline TIFF라고 부르는 기본 스펙이 존재한다. 즉 반드시 있어야 하는 tag가 존재한다. 가장 단순한 흑백 이미지를 만든다.

## Strip/Tile

---

이미지가 1920 * 1080 RGB라고 한다면, 전체 픽셀 데이터는 1920 * 1080 * 3 = 약 6MB이다. 이걸 파일에 저장하는 방법은 두 가지가 있다.

1. 통째로 저장 - 단순하지만, 이미지 중간부터 읽고 싶어도 전체를 다 읽어야 한다.
2. 쪼개서 저장 - 이미지를 부분적으로 읽고 싶은 부분만 읽을 수 있다.

strip은 이미지를 가로 방향으로 몇 행씩 잘라서 저장하는 것이다. 예를 들어 1080 행짜리 이미지를 RowsPerStrip = 360으로 설정하면 360행씩 3개의 strip으로 나뉜다.

각 strip은 파일에서 다른 위치에 저장될 수 있다. 또한 strip으로 쪼개서 저장하게 되면 압축은 전체 파일에 대해서 적용되는 것이 아니라 각각의 strip에 대해서 개별적으로 적용된다. 따라서 이 strip들이 각각 어디에 있고, 각 크기가 얼마인지를 알려주는 tag가 존재한다. strip 관련 tag들:

- **273 (stripOffsets)**: 각 strip의 파일 내 위치
- **278 (RowsPerStrip)**: 한 strip에 몇 행이 들어가는지
- **279 (StripByteCounts)**: 각 strip의 바이트 크기 - 압축되면서 strip 마다 크기가 달라질 수 있다

tile은 행과 열로 나누는 것이다. 따라서 strip/tile은 동시에 적용되지 않고 하나만 적용된다. tile 관련 tag들:

- **322 (TileWidth)**: 타일의 가로 크기
- **323 (TileLength)**: 타일의 세로 크기
- **324 (TileOffsets)**: 각 tile의 파일 내 위치
- **325 (TileByteCounts)**: 각 tile의 바이트 크기

압축은 Compression tag(259) 값에 따라 다른 코덱이 적용된다. 주요 압축 방식:

- **LZW = 5**
- **JPEG = 7**
- **Deflate = 8**
- **PackBits = 32773**

## TIFF 특징

---

- TIFF는 오프셋 기반 구조로 설계되어 있다. 따라서 데이터 배치 순서가 자유롭다.
- 이미지 하나만을 저장하는 것이 아닌 여러 이미지를 저장하는 이미지 컨테이너 형식이다.
- 이미지 유형별로 필수 tag들이 정해져 있다.
- 오프셋이 절대 위치 기반이다. 4바이트 오프셋이므로 파일의 최대 크기가 4GB로 제한된다.
- 이를 해결하기 위해서 오프셋이 8바이트 기반인 BigTIFF(magic 43)가 존재한다.

# libtiff 분석

## 기본 파일 구조

---

서브 디렉토리 없이 `.c`, `.h` 파일들이 병렬적으로 나열된 구조이다.

### 헤더 파일

- `tiff.h` - tag ID, 상수 정의
- `tiffio.h` - 외부 API 선언
- `tiffiop.h` - 내부 구조체/함수 선언

### 핵심 로직 (TIFF 파싱 흐름)

- `tif_open.c` → `tif_dirread.c` → `tif_dir.c` → `tif_read.c`

### 압축 코덱 (tif_코덱이름.c 패턴)

- `tif_lzw.c`, `tif_jpeg.c`, `tif_zip.c`, `tif_packbits.c`, `tif_fax3.c` 등

### 유틸리티

- `tif_swab.c`, `tif_strip.c`, `tif_tile.c`, `tif_print.c` 등

### 플랫폼별 코드

- `tif_unix.c`, `tif_win32.c`

## TIFF 구조체 정리

---

TIFF 파일의 모든 상태를 담고 있는 핵심 구조체이다. tiffiop.h에 선언되어 있다.

### 파일 기본 정보

- `tif_name` - 파일 이름
- `tif_fd` - 파일 디스크립터
- `tif_mode` - 읽기/쓰기 모드
- `tif_flags` - 상태 플래그 (TIFF_SWAB, TIFF_BIGTIFF, TIFF_ISTILED, TIFF_MAPPED 등)

### 헤더

- `tif_header` - 파일 헤더 (byte order, magic number, 첫 IFD offset). ClassicTIFF/BigTIFF 공용 union

### IFD 관리

- `tif_diroff` - 현재 IFD의 파일 내 위치
- `tif_nextdiroff` - 다음 IFD 위치
- `tif_curdir` - 현재 몇 번째 IFD인지 (인덱스)
- `tif_dir` - 현재 IFD의 tag 값들을 메모리에 파싱해서 담은 구조체 (`TIFFDirectory` 타입)
    - `TIFFDirectory` - 현재 IFD의 tag 값들을 저장하는 구조체

### 코덱 함수 포인터

- `tif_decoderow` / `tif_encoderow` - scanline 단위 디코딩/인코딩
- `tif_decodestrip` / `tif_encodestrip` - strip 단위
- `tif_decodetile` / `tif_encodetile` - tile 단위

압축 방식 (Compression tag)에 따라 이 포인터들이 다른 함수를 가리킨다.
ex) LZW면 LZW 디코더, JPEG면 JPEG 디코더

### I/O 버퍼

- `tif_rawdata` - 원시 데이터 버퍼
- `tif_rawdatasize` - 버퍼 크기
- `tif_rawcp` - 버퍼 내 현재 읽기/쓰기 위치

### I/O 콜백

- `tif_readproc` / `tif_writeproc` - 파일 읽기/쓰기
- `tif_seekproc` - 파일 탐색
- `tif_closeproc` - 파일 닫기

### Tag 지원

- `tif_fields` - 등록된 tag 정의 테이블
- `tif_tagmethods` - tag get/set/print 함수 포인터

## API 함수 정리

---

### 파일 열기/닫기

- `TIFFOpen` - 파일 열기 (헤더 파싱, 첫 IFD 로드)
- `TIFFClose` - 파일 닫기, 리소스 해제

### IFD 탐색

- `TIFFReadDirectory` - 현재 오프셋의 IFD를 읽어서 `tif_dir` 에 저장
- `TIFFSetDirectory` - n번째 IFD로 이동
- `TIFFWriteDirectory` - 현재 IFD를 파일에 쓰기

### Tag 값 접근

- `TIFFGetField` - tag 값 읽기
- `TIFFSetField` - tag 값 쓰기

### 이미지 데이터 읽기

- `TIFFReadScanline` - 한 행씩 읽기
- `TIFFReadEncodedStrip` - strip 단위로 읽기 (디코딩 포함)
- `TIFFReadEncodedTile` - tile 단위로 읽기 (디코딩 포함)
- `TIFFReadRawStrip` - strip 원시 데이터 읽기 (디코딩 없이)
- `TIFFReadRGBAImage` - 이미지를 RGBA로 변환해서 한 번에 읽기

### 이미지 데이터 쓰기

- `TIFFWriteScanline` - 한 행씩 쓰기
- `TIFFWriteEncodedStrip` - strip 단위로 쓰기
- `TIFFWriteEncodedTile` - tile 단위로 쓰기

### 유틸리티

- `TIFFIsTiled` - strip기반인지 tile 기반인지 확인
- `TIFFIsBigEndian` - 빅 엔디안 파일인지 확인
- `TIFFIsBigTIFF` - BigTIFF인지 확인
- `TIFFCurrentDirectory` - 현재 IFD 인덱스 반환

# 참고 문헌

## TIFF

---

### TIFF 파일 포맷

- TIFF 6.0 스펙 PDF: [https://download.osgeo.org/libtiff/doc/TIFF6.pdf](https://download.osgeo.org/libtiff/doc/TIFF6.pdf) — Header, IFD, Tag Entry, Strip/Tile, Data Type, 압축 방식 등
- libtiff 공식 문서의 BigTIFF Design 페이지: [https://libtiff.gitlab.io/libtiff/specification/bigtiff.html](https://libtiff.gitlab.io/libtiff/specification/bigtiff.html)

### libtiff

- 공식 문서: [https://libtiff.gitlab.io/libtiff/](https://libtiff.gitlab.io/libtiff/)
- 소스 코드(GitLab): [https://gitlab.com/libtiff/libtiff](https://gitlab.com/libtiff/libtiff)