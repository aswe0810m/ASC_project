# 이화진 (libtiff, libpng, libjpeg-turbo 분석 보고서)

분석자: 이화진

## 1. 개요

이미지 처리 분야에서 글로벌 표준(De-facto Standard)으로 사용되는 3대 오픈소스 라이브러리(`libtiff`, `libpng`, `libjpeg-turbo`)의 핵심 동작 매커니즘을 분석하고, 공식 가이드를 바탕으로 작성된 실전 구현 코드를 제공합니다.

## 2. 라이브러리별 소스 출처 및 핵심 코드

### 2.1 libtiff (TIFF 포맷 처리)

- **코드 출처:** `libtiff` 공식 문서의 [Using The TIFF Library](http://www.simplesystems.org/libtiff/libtiff.html) 가이드 및 소스 저장소 내 `tools/` 예제 프로토타입 기반.
- **특징:** 태그(Tag) 구조 분석 및 `TIFFReadRGBAImage`를 활용한 자동 픽셀 변환.

```c
#include <stdio.h>
#include <stdlib.h>
#include <tiffio.h>

int main(){
    // 1. TIFF 파일 스트림 오픈
    TIFF* tif = TIFFOpen("input.tif", "r");
    if (!tif) {
        fprintf(stderr, "TIFF 파일 오픈 실패\n");
        return 1;
    }

    uint32 width, height;

    // 2. 태그(Tag) 정보를 이용해 이미지 가로, 세로 크기 획득
    TIFFGetField(tif, TIFFTAG_IMAGEWIDTH, &width);
    TIFFGetField(tif, TIFFTAG_IMAGELENGTH, &height);

    // 3. RGBA 픽셀 데이터를 담을 메모리 할당 (_TIFFmalloc 사용 권장)
    uint32* raster = (uint32*) _TIFFmalloc(width * height * sizeof(uint32));
    if (!raster) {
        TIFFClose(tif);
        return 1;
    }

    // 4. 복잡한 내부 압축을 자동으로 풀어 표준 RGBA 배열로 로드
    if (TIFFReadRGBAImage(tif, width, height, raster, 0)) {
        printf("[libtiff] 로드 성공: %d x %d\n", width, height);
    }

    // 5. 자원 반환
    _TIFFfree(raster);
    TIFFClose(tif);
    return 0;
}
```

### 2.2 libpng (PNG 포맷 처리)

- **코드 출처:** `libpng` 공식 소스 트리에 포함된 표준 예제 파일 [`example.c`](https://www.google.com/search?q=https://github.com/pnggroup/libpng/blob/libpng16/example.c) 및 `libpng` 매뉴얼 가이드 기반.
- **특징:** `setjmp` 구조를 통한 하드웨어/소프트웨어 에러 핸들링 및 행(Row) 패킹 메모리 구조.

```c
#include <stdio.h>
#include <stdlib.h>
#include <png.h>

int main(){
    FILE *fp = fopen("input.png", "rb");
    if (!fp) return 1;

    // 1. 핵심 구조체 할당
    png_structp png_ptr = png_create_read_struct(PNG_LIBPNG_VER_STRING, NULL, NULL, NULL);
    png_infop info_ptr = png_create_info_struct(png_ptr);

    // 2. 예외 처리(Error Handling) 빌드 - libpng 필수 규격
    if (**setjmp(png_jmpbuf(png_ptr)**)) {
        png_destroy_read_struct(&png_ptr, &info_ptr, NULL);
        fclose(fp);
        return 1;
    }

    // 3. I/O 초기화 및 헤더 읽기
    png_init_io(png_ptr, fp);
    png_read_info(png_ptr, info_ptr);

    int width       = png_get_image_width(png_ptr, info_ptr);
    int height      = png_get_image_height(png_ptr, info_ptr);
    int color_type  = png_get_color_type(png_ptr, info_ptr);
    int bit_depth   = png_get_bit_depth(png_ptr, info_ptr);

    // 4. 다양한 PNG 포맷(팔레트, 그레이스케일 등)을 표준 RGB/RGBA로 트랜스폼 설정
    if (color_type == PNG_COLOR_TYPE_PALETTE) png_set_palette_to_rgb(png_ptr);
    if (color_type == PNG_COLOR_TYPE_GRAY && bit_depth < 8) png_set_expand_gray_1_2_4_to_8(png_ptr);
    if (png_get_valid(png_ptr, info_ptr, PNG_INFO_tRNS)) png_set_tRNS_to_alpha(png_ptr);
    png_read_update_info(png_ptr, info_ptr);

    // 5. PNG 특유의 행(Row) 단위 2차원 포인터 배열 메모리 할당 및 디코딩
    png_bytep* row_pointers = (png_bytep*) malloc(sizeof(png_bytep) * height);
    for (int y = 0; y < height; y++) {
        row_pointers[y] = (png_byte*) malloc(png_get_rowbytes(png_ptr, info_ptr));
    }

    png_read_image(png_ptr, row_pointers);
    printf("[libpng] 로드 성공: %d x %d\n", width, height);

    // 6. 메모리 해제
    for (int y = 0; y < height; y++) free(row_pointers[y]);
    free(row_pointers);
    png_destroy_read_struct(&png_ptr, &info_ptr, NULL);
    fclose(fp);
    return 0;
}
```

- setjmp 구조
    1. **(`setjmp`):**`if (setjmp(...))` 코드가 처음 실행될 때는 **`0`을 반환**합니다. 따라서 처음에는 `if`문 블록 내부를 실행하지 않고 그대로 통과하여 다음 단계(이미지 읽기)로 넘어갑니다.
    2. **에러 발생 시 워프 (`longjmp`):**
    이후 이미지 데이터를 읽다가 파일이 깨졌거나 메모리가 부족한 에러가 발생하면, `libpng` 내부에서 `longjmp`를 호출합니다.
    3. **예외 처리 실행:**`longjmp`가 실행되면 프로그램은 아까 세워둔 `setjmp` 위치로 번지점프하듯 되돌아옵니다. 이때는 `setjmp`가 **`0`이 아닌 값**을 반환하기 때문에, `if`문 안으로 들어가서 안전하게 메모리를 해제(`png_destroy_read_struct`)하고 프로그램을 종료(`return 1`)하게 됩니다.

### 2.3 libjpeg-turbo (JPEG 포맷 처리)

- jpeg : 컬러 순간 동작(steal) 이미지를 위한 국제적인 압축표준, JPEG는 이미지를 작은 블록으로 나누어 많은 양의 이미지 정보를 줄이는 DCT(Discrete Cosine Transformer) 알고리즘에 기초
- **특징:** 루프를 활용한 고속 스캔라인 디코딩 및 SIMD 가속 최적화 파이프라인.
- 설명
    
    <aside>
    
    SIMD 명령( MMX, SSE2, NEON)을 사용해서 빠르게 인코딩/디코딩하는 JPEG 이미지 코덱이다.
    
    - 스캔라인 : 이미지의 가로 한 줄
    코드의 while 루프 문을 **가로 한 줄(스캔라인)씩 순차적으로 읽어 내리며** 디코딩
    - **SIMD :** CPU가 "하나의 명령어로 여러 개의 데이터를 동시에 처리"하는 하드웨어 가속 기술, SIMD 가속이 적용되면 **한 번에 8개, 16개씩 픽셀을 묶어서 동시에** 계산
    
    파일에서 압축된 데이터를 읽어오기 → SIMD로 초고속 수학 연산하기 → 1차원 연속 플랫 버퍼(raw_image)에 한 줄씩 채워 넣기
    
    즉 하나의 명령어를 실행하는 동안 같은 오퍼레이션이 병렬로 데이터를 처리하여 속도가 빠른 것이다.
    
    </aside>
    
- **코드 출처:** `libjpeg-turbo` 공식 저장소의 예제 파일 [`djpeg.c`](https://www.google.com/search?q=https://github.com/libjpeg-turbo/libjpeg-turbo/blob/main/djpeg.c) 및 IJG(Independent JPEG Group) 표준 API 가이드 소스 기반. [https://libjpeg-turbo.org/](https://libjpeg-turbo.org/) https://github.com/libjpeg-turbo/libjpeg-turbo

```c
/* libjpeg-turb 라이브러리를 사용하여 JPEG 이미지 파일(input.jpg)의 압축을 풀고, 컴퓨터가 처리할 수 있는 순수한 픽셀 데이터(RGB)로 변환해 메모리에 로드하는 디코딩(Decompress) 프로그램 */

#include <stdio.h>
#include <stdlib.h>
#include <jpeglib.h>

int main(){
    FILE *infile = fopen("input.jpg", "rb");
    if (!infile) return 1;

    // 1. 디코더 및 에러 핸들러 객체 초기화
    struct jpeg_decompress_struct cinfo;
    struct jpeg_error_mgr jerr;

    cinfo.err = jpeg_std_error(&jerr);
    jpeg_create_decompress(&cinfo);

    // 2. 소스 스트림 지정 및 헤더 데이터 read
    jpeg_stdio_src(&cinfo, infile);
    jpeg_read_header(&cinfo, TRUE);

    // 3. 디코딩 프로세스 시작
    jpeg_start_decompress(&cinfo);

    int width = cinfo.output_width;
    int height = cinfo.output_height;
    int num_components = cinfo.output_components; // RGB = 3

    // 연속된 1차원 거대 버퍼 할당
    unsigned char* raw_image = (unsigned char*)malloc(width * height * num_components);

    // 4. 스캔라인(Scanline) 단위 루프 수행 (이 구간에서 SIMD 가속 작동)
    JSAMPROW row_pointer[1];
    while (cinfo.output_scanline < cinfo.output_height) {
        // 버퍼의 현재 행 시작 주소 계산 후 바인딩
        row_pointer[0] = &raw_image[cinfo.output_scanline * width * num_components];
        jpeg_read_scanlines(&cinfo, row_pointer, 1);
    }

    printf("[libjpeg-turbo] 로드 성공: %d x %d\n", width, height);

    // 5. 디코더 종료 및 메모리 반환
    jpeg_finish_decompress(&cinfo);
    jpeg_destroy_decompress(&cinfo);
    free(raw_image);
    fclose(infile);

    return 0;
}
```

```cpp
/* 이미지 로드 */

// 외부에서 호출하는 간결한 인터페이스 함수
bool JpegCtrl::Load(wstring filepath, ImageEx& image, int _channels){
	jpeg_decompress_struct cinfo; // JPEG 디압축 구조체 선언
	
	/* 디압축 구조체란
	libjpeg-turbo 라이브러리에서 JPEG 이미지의 압축을 푸는(Decompress)
	모든 과정과 상태, 설정 값을 담아두는 핵심 컨텍스트 객체
	*/

	// 실제 복잡한 해제 로직을 가진 내부 함수를 호출하여 결과를 반환
	return read_jpeg_file(cinfo, filepath, image);
}

// 실제 디압축(디코딩)을 수행하는 함수 
// 디압축 구조체 정의 : 이미지의 가로, 세로 크기, 컬러 수 정보
int JpegCtrl::read_jpeg_file(jpeg_decompress_struct& cinfo, wstring& filepath, ImageEx& image){
	my_error_mgr jerr;          // 사용자 정의 에러 매니저 (setjmp 버퍼 포함)
	FILE* infile;               // 소스 파일 포인터
	unsigned char* rowptr[1];   // 디코딩된 한 행(Scanline)의 데이터가 저장될 메모리 주소 배열
	int row_stride;             // 출력 버퍼의 물리적 행 너비 (현재 코드에선 미사용 주석 처리)

	// 유니코드 파일 경로(wstring)를 지원하는 안전한 파일 열기 (바이너리 읽기 모드)
	_wfopen_s(&infile, filepath.c_str(), L"rb");
	if (infile == nullptr)
		return false; // 파일 열기 실패 시 즉시 종료

	// Step 1. JPEG 디압축 객체 할당 및 에러 핸들러 초기화
	cinfo.err = jpeg_std_error(&jerr.pub);       // 기본 에러 루틴 세팅
	jerr.pub.error_exit = JpegCtrl::my_error_exit; // 커스텀 에러 콜백 함수 등록
	
	// libjpeg 내부에서 에러(예: 파일 손상) 발생 시 my_error_exit 내부의 longjmp를 통해 이리로 점프함
	if (setjmp(jerr.setjmp_buffer))
	{
		// 에러 발생 시 자원을 안전하게 해제하고 0(실패) 반환
		jpeg_destroy_decompress(&cinfo);
		fclose(infile);
		return 0;
	}
	jpeg_create_decompress(&cinfo); // 디압축 구조체 공식 초기화

	// Step 2. 데이터 소스 지정 (C 표준 파일 스트림 사용)
	jpeg_stdio_src(&cinfo, infile);

	// Step 3. jpeg_read_header()를 호출하여 파일의 헤더 정보(크기, 채널 등)를 읽음
	(void)jpeg_read_header(&cinfo, TRUE);

	// Step 4. 필요한 경우 여기서 디압축 매개변수 설정 가능 (현재는 기본값 사용으로 생략)

	// Step 5. 디압축 디코더 시작
	(void)jpeg_start_decompress(&cinfo);
	
	// 헤더 및 start 완료 후 채워진 가로, 세로, 채널 정보를 가져옴
	int nCols = cinfo.output_width;     // 이미지 가로 픽셀 수
	int nRows = cinfo.output_height;    // 이미지 세로 픽셀 수
	int nChannels = cinfo.num_components; // 채널 수 (보통 RGB는 3)
	
	// 가져온 정보로 데이터를 담을 ImageEx 메모리 공간 할당 (BGR 포맷 지정)
	image.Create(nRows, nCols, nChannels, ColorFormat::ColorFormatBGR);

	// Step 6. 이미지의 모든 행(Scanline)을 읽을 때까지 반복 실행
	// cinfo.output_scanline은 0부터 시작하여 행을 읽을 때마다 자동으로 1씩 증가함
	while (cinfo.output_scanline < cinfo.output_height)
	{
		// image.Get(0): 이미지 데이터 버퍼의 첫 시작 주소
		// image.WidthStep() * cinfo.output_scanline: 현재 채워야 하는 행의 바이트 오프셋 위치
		// 디코딩된 결과물이 저장될 정확한 메모리 주소를 rowptr[0]에 대입
		rowptr[0] = image.Get(0) + image.WidthStep() * cinfo.output_scanline;
		
		// libjpeg 라이브러리가 1개의 행을 디코딩하여 rowptr[0] 주소에 직접 작성함
		(void)jpeg_read_scanlines(&cinfo, rowptr, 1);
	}

	// Step 7. 디압축 작업 완료 선언
	(void)jpeg_finish_decompress(&cinfo);

	// Step 8. 사용이 끝난 JPEG 디압축 객체 메모리 해제 및 파일 닫기
	jpeg_destroy_decompress(&cinfo);
	fclose(infile);

	return 1; // 성공적으로 로드 완료 (true)
}
```

```cpp
/* 이미지 저장 */

bool JpegCtrl::Save(wstring filepath, ImageEx& image, int quality){
	jpeg_compress_struct cinfo; // JPEG 압축 구조체 (압축 환경 및 상태 관리)
	jpeg_error_mgr jerr;        // 에러 핸들러 구조체

	FILE* outfile;              // 저장할 대상 파일 포인터
	unsigned char* rowptr[1];   // libjpeg에 전달할 한 행(Row)의 시작 주소 배열

	// Step 1. JPEG 압축 객체 할당 및 초기화
	cinfo.err = jpeg_std_error(&jerr); // 기본 에러 처리 루틴 설정
	jpeg_create_compress(&cinfo);      // 압축 구조체 초기화

	// Step 2. 데이터가 저장될 목적지(파일) 지정
	_wfopen_s(&outfile, filepath.c_str(), L"wb"); // 유니코드 경로 파일 생성 (바이너리 쓰기 모드)
	if (outfile == nullptr)
	{
		// 파일 생성 실패 시 초기화했던 자원을 해제하고 종료해야 메모리 누수가 없습니다.
		jpeg_destroy_compress(&cinfo); 
		return false;
	}
	jpeg_stdio_dest(&cinfo, outfile); // libjpeg의 출력 대상을 방금 연 파일로 지정

	// Step 3. 압축을 위한 필수 매개변수(이미지 정보) 설정
	cinfo.image_width = image.Cols();          // 이미지의 가로 픽셀 크기
	cinfo.image_height = image.Rows();         // 이미지의 세로 픽셀 크기
	cinfo.input_components = image.Channels(); // 픽셀당 채널 수 (예: RGB는 3, 그레이스케일은 1)
	cinfo.in_color_space = JCS_RGB;            // 입력할 이미지의 색상 공간 (RGB 지정)
	
	jpeg_set_defaults(&cinfo);                // 설정한 정보를 바탕으로 기본 압축 매개변수 적용
	jpeg_set_quality(&cinfo, quality, TRUE);  // JPEG 압축 품질 설정 (0 ~ 100 사이의 값)

	// Step 4. 압축 프로세스 시작
	jpeg_start_compress(&cinfo, TRUE);

	// Step 5. 모든 행(Scanline)을 순서대로 파일에 기록
	// cinfo.next_scanline은 0부터 시작하여 행을 기록할 때마다 자동으로 1씩 증가합니다.
	while (cinfo.next_scanline < cinfo.image_height)
	{
		// image.Get(0): 이미지 데이터의 첫 시작 주소
		// image.WidthStep() * cinfo.next_scanline: 현재 기록해야 하는 행의 바이트 오프셋
		// rowptr[0]이 현재 다루어야 할 행의 메모리 시작 주소를 가리키게 합니다.
		rowptr[0] = image.Get(0) + image.WidthStep() * cinfo.next_scanline;
		
		// 지정한 1개의 행(Scanline) 데이터를 압축하여 파일에 씁니다.
		(void)jpeg_write_scanlines(&cinfo, rowptr, 1);
	}

	// Step 6. 압축 프로세스 공식 완료 및 파일 닫기
	jpeg_finish_compress(&cinfo);
	fclose(outfile);

	// Step 7. 사용이 끝난 JPEG 압축 객체 메모리 해제
	jpeg_destroy_compress(&cinfo);

	return true;
}
```

## 3. 종합 비교 분석

| **비교 항목** | **libtiff** | **libpng** | **libjpeg-turbo** |
| --- | --- | --- | --- |
| **공식 소스 기반** | `tif_open.c` 구조 인터페이스 | `example.c` 에러/변환 파이프라인 | `djpeg.c` 스캔라인 루프 인터페이스 |
| **메모리 할당 방식** | 전용 드라이버 래퍼 (`_TIFFmalloc`) | 행 단위 포인터 배열 구조 (`png_bytep*`) | 1차원 연속 플랫 버퍼 (`unsigned char*`) |
| **핵심 매커니즘** | **태그 기반 검색**: 이미지 메타데이터 유연성 극대화 | **청크 기반 순차 검증**: 데이터 손실 없는 무결성 중심 | **스트리밍 디코딩**: SIMD 하드웨어 가속을 통한 초고속 처리 |
| **주요 예외 처리** | 내부 리턴값 핸들링 | `setjmp` / `longjmp` 기반의 시그널 방식 | `jpeg_error_mgr` 기반 구조체 전달 |

*전용 드라이버 래퍼 : 
특정 라이브러리나 하드웨어 드라이버가 내부적으로 안전하고 일관되게 메모리를 관리하기 위해 일반적인 메모리 할당 함수(`malloc`)를 한 번 더 감싸서(Wrap) 만든 전용 함수

*1차원 연속 플랫 버퍼 : 메모리 상에 구멍이나 끊김 없이, 한 줄로 길게 쭉 늘어선 바이트(Byte) 배열

*jpeg_error_mgr 기반 구조체 전달 : `libjpeg-turbo`의 기본 에러 지침서