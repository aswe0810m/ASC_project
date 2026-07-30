# function

분석자: 유창하

## 실제 중심 함수 요약

| 기능 | libpng | libjpeg / TurboJPEG | libtiff |
| --- | --- | --- | --- |
| 헤더·구조 확정 | `png_read_info` | `jpeg_read_header`, `tj3DecompressHeader` | `TIFFOpen` → `TIFFReadDirectory` |
| 출력 geometry 확정 | `png_read_update_info` | `jpeg_calc_output_dimensions`, `jpeg_start_decompress` | `TIFFScanlineSize`, `TIFFStripSize`, `TIFFTileSize` |
| 행 디코딩 worker | `png_read_row` | `jpeg_read_scanlines` | `TIFFReadScanline` |
| 전체 이미지 wrapper | `png_read_image` | `tj3Decompress8` | `TIFFReadRGBAImage` |
| 부분·가장자리 | `png_read_row`의 Adam7 처리 | `jpeg_crop_scanline`, `tj3SetCroppingRegion` | `TIFFReadRGBATileExt` |
| 점진 처리 | `png_process_data` | `jpeg_consume_input` + buffered output | 직접 대응 없음 |
| 행 인코딩 worker | `png_write_row` | `jpeg_write_scanlines` | `TIFFWriteScanline` |
| 전체 인코딩 wrapper | `png_write_image` | `tj3Compress8` | strip/tile write 반복 |

## 1. 헤더·이미지 구조 읽기

### libpng: `png_read_info`가 중심

```
png_create_read_struct
→ png_create_info_struct
→ png_init_io / png_set_read_fn
→ png_read_info                         pngread.c:93
   ├─ png_read_sig                      pngrutil.c:116
   │  └─ png_read_data                  pngrio.c:31
   ├─ png_read_chunk_header 반복        pngrutil.c:183
   └─ png_handle_chunk                  pngrutil.c:3107
      └─ IHDR: png_handle_IHDR          pngrutil.c:901
         → png_set_IHDR                 pngset.c:435
         → png_check_IHDR               png.c:1961
```

`png_read_info`는 첫 IDAT header에서 멈춘다. signature, chunk length·순서,
IHDR width/height와 사용자 limit를 확인하는 chunk 상태 머신이다.

### libjpeg: `jpeg_read_header`가 중심

```
jpeg_create_decompress
→ jpeg_stdio_src / jpeg_mem_src / custom source manager
→ jpeg_read_header                      src/jdapimin.c:267
   └─ jpeg_consume_input                src/jdapimin.c:313
      ├─ src->init_source
      └─ consume_markers                src/jdinput.c:331
         └─ read_markers                src/jdmarker.c:968
            ├─ get_sof                  src/jdmarker.c:242
            ├─ get_sos                  src/jdmarker.c:308
            ├─ get_dht                  src/jdmarker.c:437
            └─ get_dqt                  src/jdmarker.c:511
→ initial_setup                         src/jdinput.c:47
→ default_decompress_parms              src/jdapimin.c:130
→ DSTATE_READY
```

marker 길이, SOF component allocation, sampling/block geometry가 주요 검증
지점이다.

### libtiff: `TIFFOpen`과 `TIFFReadDirectory`가 한 흐름

```
TIFFOpen                                tif_unix.c:233
→ TIFFClientOpenExt                     tif_open.c:300
   ├─ I/O callback 저장
   ├─ TIFF/BigTIFF header 검증           tif_open.c:555
   └─ TIFFReadDirectory                 tif_dirread.c:4328
      ├─ _TIFFCheckDirNumberAndOffset   tif_dirread.c:5765
      ├─ TIFFFetchDirectory             tif_dirread.c:6078
      ├─ TIFFFetchNormalTag             tif_dirread.c:6374
      ├─ TIFFSetField                   tif_dir.c:1167
      └─ TIFFSetCompressionScheme       tif_compress.c:179
         └─ codec callback 설치
```

TIFF는 `TIFFOpen`이 첫 IFD를 읽는다. 이후 공개 `TIFFReadDirectory`는 다음
IFD로 이동한다. Compression tag가 `tif_decoderow`, `tif_decodestrip`,
`tif_decodetile` 같은 function pointer를 설치하는 것이 PNG/JPEG와 다른
구조적 특징이다.

## 2. 출력 크기와 buffer 계약

### libpng

`png_get_rowbytes`가 계산 중심이 아니다. 실제 geometry barrier는 변환 설정
후 호출하는 `png_read_update_info`다.

```
png_read_info
→ png_set_expand / png_set_gray_to_rgb / png_set_filler 등
→ png_read_update_info                  pngread.c:172
   ├─ png_read_start_row                pngrutil.c:4408
   │  ├─ png_init_read_transformations  pngrtran.c:1424
   │  ├─ maximum_pixel_depth 계산
   │  └─ row/previous-row buffer 할당
   └─ png_read_transform_info           pngrtran.c:2069
      └─ 최종 info_ptr->rowbytes        pngrtran.c:2265-2276
→ png_get_rowbytes                      pngget.c:40
   └─ 계산된 값을 반환하는 leaf getter
```

### libjpeg

```
jpeg_calc_output_dimensions             src/jdmaster.c:264
├─ jpeg_core_output_dimensions           src/jdmaster.c:96
├─ component downsampled width/height
└─ output_width/height/components 갱신

jpeg_start_decompress
└─ master_selection                      src/jdmaster.c:511
   ├─ 위 계산을 다시 수행
   └─ output_width × components 표현 가능성 검사  :535-540
```

이 API는 byte size를 반환하거나 buffer를 할당하지 않는다. caller가 확정된
`output_width × output_components`로 scanline 크기를 계산해야 한다.

### TurboJPEG와 libtiff

- `tj3JPEGBufSize`: **인코딩된 JPEG destination 상한**이다.
- `tj3YUVBufSize`/`tj3YUVPlaneSize`: planar YUV buffer 크기다.
- RGB decode destination은 width·height·pixel format·pitch로 caller가 계산한다.
- `TIFFScanlineSize`: scanline decoded byte 수다.
- `TIFFStripSize`: 일반 strip 최대 크기이며 마지막 strip 실제 크기는
`TIFFReadEncodedStripGetStripSize`가 다시 계산한다.
- `TIFFTileSize`: 가장자리에서도 full coded tile 크기다. 논리 유효 폭·높이는
포함하지 않는다.

따라서 `png_get_rowbytes`, `jpeg_calc_output_dimensions`, `tj3JPEGBufSize`,
`TIFFTileSize`를 하나의 동일 대응 API로 취급하면 안 된다.

## 3. 행 단위 디코딩

### libpng data-plane: `png_read_row`

```
png_read_row                           pngread.c:288
├─ png_read_start_row                  pngrutil.c:4408
├─ current Adam7 iwidth로 rowbytes 계산
├─ png_read_IDAT_data                  pngrutil.c:4172
│  └─ PNG_INFLATE                     pngrutil.c:4256
├─ png_read_filter_row                 pngrutil.c:4154
├─ png_do_read_transformations         pngrtran.c:4880
├─ png_do_read_interlace               pngrutil.c:3711
├─ png_combine_row                     pngrutil.c:3227
└─ png_read_finish_row                 pngrutil.c:4357
```

`png_read_row`에는 caller buffer length 인자가 없다. 변환 후 rowbytes,
Adam7 부분 폭과 caller allocation의 일치가 핵심 계약이다.

### libjpeg data-plane: `jpeg_read_scanlines`

```
jpeg_read_scanlines                    src/jdapistd.c:319
└─ main->_process_data                 src/jdapistd.c:359
   ├─ process_data_simple_main         src/jdmainct.c:296
   │  └─ coef->_decompress_data
   │     └─ decompress_onepass         src/jdcoefct.c:86
   │        ├─ entropy->decode_mcu
   │        └─ inverse_DCT
   └─ post->_post_process_data
      └─ sep_upsample                  src/jdsample.c:62
         ├─ 마지막 출력 row 제한       src/jdsample.c:89-98
         └─ color_convert
            └─ ycc_rgb_convert         src/jdcolor.c:262
               └─ caller scanline
```

공개 `jpeg_read_scanlines`는 이미 끝났는지만 확인한다. 실제 마지막 row 수
제한은 upsampler에서 일어난다. edge MCU 유효 폭은 `decompress_onepass`의
마지막 MCU 처리에서 제한된다.

### libtiff data-plane: `TIFFReadScanline`

```
TIFFReadScanline                       tif_read.c:489
├─ TIFFCheckRead                       tif_read.c:1635
├─ TIFFSeek                            tif_read.c:326
│  ├─ row/sample 범위 검사
│  ├─ row → strip 계산
│  └─ TIFFFillStrip                    tif_read.c:817
├─ TIFFStartStrip                      tif_read.c:1535
│  ├─ tif_setupdecode
│  └─ tif_predecode
├─ tif_decoderow
└─ tif_postdecode
   └─ caller scanline
```

색상 변환은 하지 않고 codec sample 형식을 반환한다. `TIFFSeek` 실패 시에는
buffer를 0으로 채우지만, 모든 `tif_decoderow` 실패에 동일하게 적용되는 것은
아니다.

## 4. 전체 이미지 디코딩

### libpng: row worker 반복 wrapper

```
png_read_image                         pngread.c:608
├─ png_set_interlace_handling
├─ png_start_read_image
└─ pass × height loop
   └─ png_read_row
```

별도 whole-image decoder가 아니다. 결함 분석의 실제 하위 target은
`png_read_row`와 Adam7 combine 경로다.

### TurboJPEG: classic 흐름 전체를 감싼 one-shot wrapper

```
tj3Decompress8                         src/turbojpeg-mp.c:145
├─ handle/input/pitch/pixel format 검증
├─ jpeg_mem_src_tj
├─ jpeg_read_header
├─ output color space·scaling 설정
├─ jpeg_start_decompress
├─ optional jpeg_crop_scanline
├─ optional jpeg_skip_scanlines
├─ pitch 기반 row pointer 배열 구성
├─ jpeg_read_scanlines 반복
└─ jpeg_finish_decompress
```

destination byte length 인자는 없다. pitch와 crop height를 바탕으로 필요한
공간을 caller가 확보해야 한다. bottom-up은 row pointer 방향을 뒤집는다.

### libtiff: tile/strip + 색상별 put dispatch

```
TIFFReadRGBAImage                     tif_getimage.c:659
→ TIFFReadRGBAImageOriented            tif_getimage.c:634
→ TIFFRGBAImageBegin                   tif_getimage.c:311
   ├─ TIFFRGBAImageOK
   ├─ PickContigCase                   tif_getimage.c:3113
   └─ PickSeparateCase                 tif_getimage.c:3301
→ TIFFRGBAImageGet                     tif_getimage.c:590
   └─ gtTileContig / gtTileSeparate / gtStripContig / gtStripSeparate
      ├─ TIFFReadEncodedTile/Strip
      ├─ photometric별 put* 색상 변환
      └─ caller ABGR raster 기록
→ orientation 처리
→ TIFFRGBAImageEnd
```

실제 중심은 wrapper 이름보다 `gt*` reader와 선택된 `put*` routine이다.

## 5. 부분 영역과 가장자리

### PNG

`png_read_rows`는 crop API가 아니다. 현재 stream 위치부터 여러 row를 읽는
convenience wrapper다. 공간적인 가장자리 비교 대상은 `png_read_row` 내부의
Adam7 `png_do_read_interlace`와 `png_combine_row`다.

### JPEG/TurboJPEG

```
jpeg_crop_scanline                    src/jdapistd.c:186
├─ x + width를 output width와 대조
├─ x를 iMCU 경계로 내림
├─ 반환 width·output_width 조정
└─ first/last iMCU·MCU column 재계산

tj3SetCroppingRegion                  src/turbojpeg.c:2020
→ scaled iMCU 정렬 검사
→ tj3Decompress8
   → jpeg_crop_scanline 결과 재검증    src/turbojpeg-mp.c:200-214
```

`jpeg_crop_scanline`은 완성 픽셀을 잘라내는 후처리 함수가 아니라, 첫 scanline
전에 디코딩할 MCU 열 범위를 바꾸는 geometry controller다.

### TIFF와 CVE-2023-52356

```
TIFFReadRGBATileExt                   tif_getimage.c:3507
├─ tiled/size/alignment 검사
├─ TIFFRGBAImageBegin
├─ col < width && row < height 검사    tif_getimage.c:3556-3562
├─ read_xsize = min(tile_w, width-col)
├─ read_ysize = min(tile_h, height-row)
├─ TIFFRGBAImageGet
└─ partial 결과를 full tile layout으로 memmove/memset
```

CVE-2023-52356의 근본 원인은 codec이 아니라 이 wrapper에서 `width-col`과
`height-row`를 범위 검증 전에 계산한 것이다. 취약 버전에서는 underflow된
`read_ysize`가 실패 이후 partial-tile fix-up의 `memmove` source offset에
들어가 OOB read/SEGV를 만들었다. 패치는 subtraction 앞에 좌표 범위 guard를
추가했다.

## 6. Progressive 처리

### libpng push model

```
png_set_progressive_read_fn
→ input fragment마다 png_process_data        pngpread.c:50
   → png_process_some_data                    pngpread.c:108
      ├─ png_push_read_sig
      ├─ png_push_read_chunk
      └─ png_push_read_IDAT
         → png_process_IDAT_data
         → png_push_process_row
         → 사용자 info/row/end callback
```

### libjpeg coefficient-buffer model

```
jpeg_read_header
→ buffered_image = TRUE
→ jpeg_start_decompress
→ jpeg_consume_input 반복                     src/jdapimin.c:313
   ├─ consume_data: MCU를 virtual coefficient buffer에 저장
   └─ consume_markers: 다음 SOS/EOI 처리
→ jpeg_start_output(scan_number)
→ jpeg_read_scanlines
→ jpeg_finish_output
→ 다음 scan 반복
→ jpeg_finish_decompress
```

TIFF의 `TIFFReadEncodedStrip`은 독립 strip random access API다. 위 두 API와
동일한 streaming state machine으로 묶으면 안 되고, block 단위 decode라는
제한된 단계 대응만 가능하다.

## 7. 인코딩 data-plane

```
png_write_image
└─ png_write_row                      pngwrite.c:746
   ├─ png_do_write_transformations
   ├─ png_write_find_filter
   ├─ png_compress_IDAT
   └─ deflate → IDAT
```

```
jpeg_write_scanlines                  src/jcapistd.c:85
└─ main->_process_data
   └─ pre_process_data                src/jcprepct.c:136
      ├─ color conversion
      ├─ downsampling
      └─ bottom padding
   └─ compress_data                   src/jccoefct.c:143
      ├─ forward DCT
      ├─ right/bottom dummy MCU
      └─ encode_mcu_huff
```

```
TIFFWriteScanline                     tif_write.c:47
├─ TIFFWriteCheck
├─ TIFFWriteBufferSetup
├─ tif_setupencode / tif_preencode
├─ tif_encoderow
└─ TIFFAppendToStrip                  tif_write.c:805
   └─ TIFFWriteFile
```

`png_write_image`는 row 반복 wrapper, `tj3Compress8`은 classic
start/write/finish one-shot wrapper다. `TIFFWriteEncodedStrip`과
`TIFFWriteEncodedTile`은 전체 이미지 함수가 아니라 한 strile의 worker이며
caller loop가 필요하다.

## 직접 실행으로 확인한 범위

| Library | Fixture와 경계 | 확인한 중심 흐름 | 결과 |
| --- | --- | --- | --- |
| libpng | 7×5 RGBA, file/memory callback | row/full read-write, simplified read-write | ASan/UBSan 5/5 pass, fingerprint `9238342d952ba92f` |
| libjpeg | 17×13 RGB, 4:2:0 progressive | scanline encode/decode, marker/ICC, buffered output, coefficient transcode | classic scenario pass |
| TurboJPEG | 17×13 RGB, 4:2:0 | one-shot encode/decode, full-region crop 설정, YUV, transform | TurboJPEG scenario pass |
| libtiff | 19×17 RGB, 5-row strip, 16×16 tile | scanline, strip, RGBA, bottom-right 3×1 edge tile, memory callback | ASan/UBSan 4/4 pass, fingerprint `b557dc3c4f58dc27` |

현재 JPEG 하네스는 성공 상태와 API 도달을 검증하지만 decoded pixel 품질
fingerprint는 기록하지 않는다. 또한 `jpeg_crop_scanline`과
`tj3SetCroppingRegion`은 현재 full-region으로만 호출되어 non-trivial crop
검증은 아직 아니다. libpng 직접 하네스도 non-interlaced fixture이므로 Adam7
경로의 직접 경계 검증은 upstream trace에 의존한다.

## 기존 기능 매핑에서 수정해야 할 항목

1. `output_size_query`를 decoded row geometry, planar buffer size, encoded
destination bound로 분리한다.
2. `jpeg_mem_src`는 callback 입력이 아니라 memory input으로 이동한다.
3. libjpeg custom source manager는 공개 setter가 아니라 `jpeg_source_mgr`
계약으로 별도 기록한다.
4. `png_read_rows`를 spatial crop과 동일시하지 않는다.
5. `TIFFReadEncodedStrip`은 progressive state API가 아니라 block decode다.
6. `tj3GetErrorStr`은 callback 등록이 아니라 error query다.
7. planar decode는 pixel rows와 분리하고, libtiff는
`PLANARCONFIG_SEPARATE`와 sample별 tile/strip 조건을 명시한다.
8. `TIFFReadRGBATileExt`를 별도 stage map으로 등록해 CVE entry와 직접
연결한다.

## 다음 직접 검증 대상

1. Adam7 PNG의 마지막 pass·마지막 row와 변환 후 rowbytes 증가
2. JPEG의 실제 부분 crop, skip 이후 첫/마지막 scanline
3. TurboJPEG의 scaled iMCU 정렬 crop과 bottom-up pitch
4. 12/16-bit JPEG 및 raw component plane
5. TIFF의 오른쪽·아래·오른쪽 아래 RGBA tile 세 경계
6. TIFF vulnerable/fixed `TIFFReadRGBATileExt` 좌표 경계 비교
7. custom libjpeg source manager의 suspension/short-read