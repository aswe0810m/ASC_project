# JPEG 및 libjpeg-turbo 분석

분석자: 오동규

# JPEG 파일 포맷 분석

## JPEG 전체 파일 구조

---

1. Signature
2. Marker Segment
3. 주요 marker
4. 기본 구조 흐름

## Signature

---

JPEG 파일은 항상 `FF D8` 으로 시작한다. 이는 SOI (Start of Image) marker이다. 바이트 오더는 빅 엔디안으로 고정이다.

## Marker Segment

---

SOI 뒤로는 marker segment가 이어진다. 각 marker는 다음과 같은 구조로 이루어져 있다.

- **Marker (2 bytes)** - 항상 `FF` 로 시작하고 그 뒤에 marker 종류를 나타내는 1바이트가 온다.
- **Length (2 bytes)** - 해당 segment의 데이터 크기 (Marker 2바이트는 미포함, Length 2바이트는 포함)
- **Data** - marker에 따른 데이터

Marker가 항상 `FF` 로 시작하는 이유는 압축된 이미지와 구분하기 위해서이다. 압축된 이미지에서는 `FF` 바이트가 나오면 반드시 뒤에 `00` 을 붙인다.

## 주요 marker

---

- `FF D8` - **SOI (Start of Image)** - 파일 시작. Length 없음
- `FF E0` - **APP0 (JFIF 메타데이터)** - 해상도, 썸네일 등
- `FF DB` - **DQT (Define Quantization Table)** - 양자화 테이블 정의
- `FF C0` - **SOF0 (Start of Frame, Baseline)** - 이미지 기본 정보(높이, 너비, 채널 수, 샘플 정밀도)
- `FF C4` - **DHT (Define Huffman Table)** - 허프만 테이블 정의
- `FF DA` - **SOS (Start of Scan)** - 이 marker 뒤부터 실제 압축된 이미지 데이터가 옴
- `FF D9` - **EOI (End of Image)** - 파일 끝. Length 없음

## 기본 구조 흐름

---

<aside>

SOI → APP0 → DQT → SOF0 → DHT → SOS → [압축 데이터] → EOI

</aside>

SOF0에서 이미지 크기와 채널 정보를 얻고, DQT/DHT에서 디코딩에 필요한 테이블을 정의하고, SOS 이후에 실제 압축 데이터가 오는 구조이다.

- **SOI**는 반드시 맨 처음
- **EOI**는 반드시 맨 마지막
- **SOS**전에 **SOF, DQT, DHT**가 와야한다. **SOF, DQT, DHT**의 순서는 상관 없다.

## 특징

---

- Marker segment 기반 순차 구조이다.
- DCT + 양자화 과정에서 데이터가 손실되는 손실 압축 기반이다.
- 바이트 오더가 빅 엔디안으로 고정되어 있다.
- RGB가 아니라 YCbCr 색상 공간을 사용한다.
- 이미지 분할이 없고, 단일 이미지 기반이다.

# libjpeg-turbo 분석

## 기본 파일 구조

---

`src/` 디렉토리 안에 `.c`, `.h` 파일들이 병렬적으로 나열된 구조이다.

### 헤더 파일

- `jpeglib.h` - 외부 API 선언
- `jpegint.h` - 내부 구조체/함수 선언
- `jerror.h` - 에러 코드 정의
- `jmorecfg.h` - 타입 정의, 설정

### 핵심 로직 - 파일명 패턴

- `jc*` - 압축(compress) 관련
- `jd*` - 해제(decompress) 관련
- `j` - 공통

### 해제(읽기) 핵심

- `jdapimin.c` - 해제 API 초기화
- `jdapistd.c` - 해제 API 표준 흐름
- `jdmarker.c` - marker 파싱 (SOF, DQT, DHT 등)
- `jdinput.c` - 입력 제어
- `jdhuff.c` - 허프만 디코딩
- `jdcoefct.c` - DCT 계수 처리
- `jddctmgr.c` - IDCT 관리
- `jdcolor.c` - 색상 변환 (YCbCr → RGB)
- `jdsample.c` - 업샘플링
- `jdmainct.c` - 메인 버퍼 제어
- `jdpostct.c` - 후처리
- `jdmaster.c` - 해제 전체 흐름 관리
- `jdatasrc.c` - 데이터 소스 (I/O)

### 압축(쓰기) 핵심

- `jcapimin.c` - 압축 API 초기화
- `jcapistd.c` - 압축 API 표준 흐름
- `jcmarker.c` - marker 쓰기
- `jchuff.c` - 허프만 인코딩
- `jccoefct.c` - DCT 계수 처리
- `jcdctmgr.c` - DCT 관리
- `jccolor.c` - 색상 변환 (RGB → YCbCr)
- `jcsample.c` - 다운샘플링
- `jcmainct.c` - 메인 버퍼 제어
- `jcprepct.c` - 전처리
- `jcmaster.c` - 압축 전체 흐름 관리
- `jdatadst.c` - 데이터 목적지 (I/O)

### DCT/IDCT 구현

- `jfdctint.c`, `jfdctfst.c`, `jfdctflt.c` - 정수/고속/부동소수점 DCT
- `jidctint.c`, `jidctfst.c`, `jfdctflt.c`, `jidctred.c` - IDCT 변형들

### 공통

- `jcomapi.c` - 공통 API
- `jerror.c` - 에러 처리
- `jmemmgr.c` - 메모리 관리
- `jutils.c` - 유틸리티

### TurboJPEG (libjpeg-turbo 전용 API)

- `turbojpeg.c` / `turbojpeg.h` - 간소화된 고수준 API

## 구조체 정리

---

libjpeg-turbo는 구조체가 압축/해제 별로 나뉘어 있다.

`jpeg_common_fields` - 압축/해제 구조체가 공유하는 필드

- `err` - 에러 핸들러
- `mem` - 메모리 관리자
- `progress` - 진행 상황 모니터
- `client_data` - 사용자 데이터
- `global_state` - 현재 처리 단계 (호출 순서 검증용)

`jpeg_decompress_struct` - 해제(읽기)용 구조체

### I/O

- `src` - 데이터 소스

### 이미지 정보 (SOF에서 파싱)

- `image_width`, `image_height` - 이미지 크기
- `num_components` - 채널 수
- `jpeg_color_space` - 파일의 색상 공간 (보통 YCbCr)

### 출력 설정 (사용자가 지정)

- `out_color_space` - 출력 색상 공간 (보통 RGB로 변환)
- `scale_num`, `scale_denom` - 축소 비율
- `dct_method` - IDCT 알고리즘 선택
- `output_width`, `output_height` - 실제 출력 크기

### 디코딩 테이블

- `quant_tbl_ptrs[]` - 양자화 테이블 (DQT에서 파싱)
- `dc_huff_tbl_ptrs[]`, `ac_huff_tbl_ptrs[]` - 허프만 테이블 (DHT에서 파싱)

### 컴포넌트 정보

- `comp_info` - 각 컴포넌트(Y, Cb, Cr) 별 정보 (샘플링 팩터 등)

### 처리 상태

- `output_scanline` - 현재까지 출력한 행 번호
- `progressive_mode` - Progressive JPEG 여부
- `unread_marker` - 아직 처리 안 한 marker

### 서브 모듈 포인터

- `master`, `main`, `coef`, `post` - 흐름 제어
- `marker` - marker 파서
- `entropy` - 허프만/산술 디코더
- `idct` - IDCT 처리
- `upsample` - 업샘플링
- `cconvert` - 색상 변환

## API 함수 정리

---

### 초기화/정리

- `jpeg_create_decompress` - 해제용 구조체 생성
- `jpeg_create_compress` - 압축용 구조체 생성
- `jpeg_destroy_decompress` - 해제용 구조체 해제
- `jpeg_destroy_compress` - 압축용 구조체 해제

### I/O 설정

- `jpeg_stdio_src` - 파일 포인터(`File*`)를 입력 소스로 연결
- `jpeg_stdio_dest` - 파일 포인터를 출력 목적지로 연결
- `jpeg_mem_src` - 메모리 버퍼를 입력 소스로 연결

### 읽기(해제)

- `jpeg_read_header` - marker들을 파싱해서 이미지 정보 읽기 (SOF, DQT, DHT 등)
- `jpeg_start_decompress` - 디코딩 시작 준비 (출력 크기 계산, 서브 모듈 초기화)
- `jpeg_read_scanlines` - 한 행씩 디코딩해서 픽셀 데이터로 출력
- `jpeg_finish_decompress` - 디코딩 마무리

### 쓰기(압축)

- `jpeg_set_defaults` - 기본 압축 설정
- `jpeg_set_quality` - 압축 품질 설정 (0~100)
- `jpeg_start_compress` - 인코딩 시작
- `jpeg_write_scanlines` - 한 행씩 인코딩
- `jpeg_finish_compress` - 인코딩 마무리

### TurboJPEG API (libjpeg-turbo 전용)

- `tjInitDecompress` - TurboJPEG 해제 핸들 생성
- `tjDecompress2` - 이미지 전체를 한번에 해제
- `tjInitCompress` - TurboJPEG 압축 핸들 생성
- `tjCompress2` - 이미지 전체를 한 번에 압축
- `tjDestroy` - 핸들 해제

# 참고 문헌

## JPEG

---

### JPEG 파일 포맷

- ITU-T T.81 (JPEG 스펙): [https://www.w3.org/Graphics/JPEG/itu-t81.pdf](https://www.w3.org/Graphics/JPEG/itu-t81.pdf) — marker 구조, SOF, DQT, DHT, SOS, byte stuffing 등
- JFIF 스펙: [https://www.w3.org/Graphics/JPEG/jfif3.pdf](https://www.w3.org/Graphics/JPEG/jfif3.pdf)

### libjpeg-turbo

- 공식 저장소: [https://github.com/libjpeg-turbo/libjpeg-turbo](https://github.com/libjpeg-turbo/libjpeg-turbo)