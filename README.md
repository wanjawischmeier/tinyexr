# Tiny OpenEXR image library.

![Example](https://github.com/syoyo/tinyexr/blob/master/asakusa.png?raw=true)

`tinyexr` is a small, single header-only library to load and save OpenEXR (.exr) images.
`tinyexr` is written in portable C++ (no library dependency except for STL), thus `tinyexr` is good to embed into your application.
To use `tinyexr`, simply copy `tinyexr.h` into your project.

## Original repository

For an up-to-date [features list](https://github.com/syoyo/tinyexr#features), informations on [how to contribute](https://github.com/syoyo/tinyexr#todo), useful [examples](https://github.com/syoyo/tinyexr#examples) and much more, please take a look at the [original repository](https://github.com/syoyo/tinyexr) by [syoyo](https://github.com/syoyo).

## Usage

NOTE: **API is still subject to change**. See the source code for details.

Include `tinyexr.h` with `TINYEXR_IMPLEMENTATION` flag (do this only for **one** .cc file).

```cpp
#define TINYEXR_IMPLEMENTATION
#include "tinyexr.h"
```

### Compile flags

* <span style="color:red">TINYEXR_USE_MINIZ</span> As opposed to the original repo, this fork always uses the system's zlib header. Setting this flag therefore won't archieve anything.
* `TINYEXR_USE_PIZ` Enable PIZ compression support (default = 1)
* `TINYEXR_USE_ZFP` Enable ZFP compression supoort (TinyEXR extension, default = 0)
* `TINYEXR_USE_THREAD` Enable threaded loading using C++11 thread (Requires C++11 compiler, default = 0)
* `TINYEXR_USE_OPENMP` Enable OpenMP threading support (default = 1 if `_OPENMP` is defined)
  * Use `TINYEXR_USE_OPENMP=0` to force disable OpenMP code path even if OpenMP is available/enabled in the compiler.
* `NOMINMAX` You may need to [set this Preprocessor Definition](https://docs.microsoft.com/en-us/cpp/build/reference/d-preprocessor-definitions?view=msvc-170#to-set-this-compiler-option-in-the-visual-studio-development-environment) (in Visual Studio) for the project to compile successfully.

### Reading layered RGB(A) EXR file.

If you want to read EXR image with layer info (channel has a name with delimiter `.`), please use `LoadEXRWithLayer` API.

You need to know layer name in advance (e.g. through `EXRLayers` API).

```cpp
  const char* input = ...;
  const char* layer_name = "diffuse"; // or use EXRLayers to get list of layer names in .exr
  float* out; // width * height * RGBA
  int width;
  int height;
  const char* err = NULL; // or nullptr in C++11

  // will read `diffuse.R`, `diffuse.G`, `diffuse.B`, (`diffuse.A`) channels
  int ret = LoadEXRWithLayer(&out, &width, &height, input, layer_name, &err);

  if (ret != TINYEXR_SUCCESS) {
    if (err) {
       fprintf(stderr, "ERR : %s\n", err);
       FreeEXRErrorMessage(err); // release memory of error message.
    }
  } else {
    ...
    free(out); // release memory of image data
  }

```

### Loading Singlepart EXR from a file.

Scanline and tiled format are supported.

```cpp
  // 1. Read EXR version.
  EXRVersion exr_version;

  int ret = ParseEXRVersionFromFile(&exr_version, argv[1]);
  if (ret != 0) {
    fprintf(stderr, "Invalid EXR file: %s\n", argv[1]);
    return -1;
  }

  if (exr_version.multipart) {
    // must be multipart flag is false.
    return -1;
  }

  // 2. Read EXR header
  EXRHeader exr_header;
  InitEXRHeader(&exr_header);

  const char* err = NULL; // or `nullptr` in C++11 or later.
  ret = ParseEXRHeaderFromFile(&exr_header, &exr_version, argv[1], &err);
  if (ret != 0) {
    fprintf(stderr, "Parse EXR err: %s\n", err);
    FreeEXRErrorMessage(err); // free's buffer for an error message
    return ret;
  }

  // // Read HALF channel as FLOAT.
  // for (int i = 0; i < exr_header.num_channels; i++) {
  //   if (exr_header.pixel_types[i] == TINYEXR_PIXELTYPE_HALF) {
  //     exr_header.requested_pixel_types[i] = TINYEXR_PIXELTYPE_FLOAT;
  //   }
  // }

  EXRImage exr_image;
  InitEXRImage(&exr_image);

  ret = LoadEXRImageFromFile(&exr_image, &exr_header, argv[1], &err);
  if (ret != 0) {
    fprintf(stderr, "Load EXR err: %s\n", err);
    FreeEXRHeader(&exr_header);
    FreeEXRErrorMessage(err); // free's buffer for an error message
    return ret;
  }

  // 3. Access image data
  // `exr_image.images` will be filled when EXR is scanline format.
  // `exr_image.tiled` will be filled when EXR is tiled format.

  // 4. Free image data
  FreeEXRImage(&exr_image);
  FreeEXRHeader(&exr_header);
```

### Loading Multipart EXR from a file.

Scanline and tiled format are supported.

```cpp
  // 1. Read EXR version.
  EXRVersion exr_version;

  int ret = ParseEXRVersionFromFile(&exr_version, argv[1]);
  if (ret != 0) {
    fprintf(stderr, "Invalid EXR file: %s\n", argv[1]);
    return -1;
  }

  if (!exr_version.multipart) {
    // must be multipart flag is true.
    return -1;
  }

  // 2. Read EXR headers in the EXR.
  EXRHeader **exr_headers; // list of EXRHeader pointers.
  int num_exr_headers;
  const char *err = NULL; // or nullptr in C++11 or later

  // Memory for EXRHeader is allocated inside of ParseEXRMultipartHeaderFromFile,
  ret = ParseEXRMultipartHeaderFromFile(&exr_headers, &num_exr_headers, &exr_version, argv[1], &err);
  if (ret != 0) {
    fprintf(stderr, "Parse EXR err: %s\n", err);
    FreeEXRErrorMessage(err); // free's buffer for an error message
    return ret;
  }

  printf("num parts = %d\n", num_exr_headers);


  // 3. Load images.

  // Prepare array of EXRImage.
  std::vector<EXRImage> images(num_exr_headers);
  for (int i =0; i < num_exr_headers; i++) {
    InitEXRImage(&images[i]);
  }

  ret = LoadEXRMultipartImageFromFile(&images.at(0), const_cast<const EXRHeader**>(exr_headers), num_exr_headers, argv[1], &err);
  if (ret != 0) {
    fprintf(stderr, "Parse EXR err: %s\n", err);
    FreeEXRErrorMessage(err); // free's buffer for an error message
    return ret;
  }

  printf("Loaded %d part images\n", num_exr_headers);

  // 4. Access image data
  // `exr_image.images` will be filled when EXR is scanline format.
  // `exr_image.tiled` will be filled when EXR is tiled format.

  // 5. Free images
  for (int i =0; i < num_exr_headers; i++) {
    FreeEXRImage(&images.at(i));
  }

  // 6. Free headers.
  for (int i =0; i < num_exr_headers; i++) {
    FreeEXRHeader(exr_headers[i]);
    free(exr_headers[i]);
  }
  free(exr_headers);
```


Saving Scanline EXR file.

```cpp
  // See `examples/rgbe2exr/` for more details.
  bool SaveEXR(const float* rgb, int width, int height, const char* outfilename) {

    EXRHeader header;
    InitEXRHeader(&header);

    EXRImage image;
    InitEXRImage(&image);

    image.num_channels = 3;

    std::vector<float> images[3];
    images[0].resize(width * height);
    images[1].resize(width * height);
    images[2].resize(width * height);

    // Split RGBRGBRGB... into R, G and B layer
    for (int i = 0; i < width * height; i++) {
      images[0][i] = rgb[3*i+0];
      images[1][i] = rgb[3*i+1];
      images[2][i] = rgb[3*i+2];
    }

    float* image_ptr[3];
    image_ptr[0] = &(images[2].at(0)); // B
    image_ptr[1] = &(images[1].at(0)); // G
    image_ptr[2] = &(images[0].at(0)); // R

    image.images = (unsigned char**)image_ptr;
    image.width = width;
    image.height = height;

    header.num_channels = 3;
    header.channels = (EXRChannelInfo *)malloc(sizeof(EXRChannelInfo) * header.num_channels);
    // Must be (A)BGR order, since most of EXR viewers expect this channel order.
    strncpy(header.channels[0].name, "B", 255); header.channels[0].name[strlen("B")] = '\0';
    strncpy(header.channels[1].name, "G", 255); header.channels[1].name[strlen("G")] = '\0';
    strncpy(header.channels[2].name, "R", 255); header.channels[2].name[strlen("R")] = '\0';

    header.pixel_types = (int *)malloc(sizeof(int) * header.num_channels);
    header.requested_pixel_types = (int *)malloc(sizeof(int) * header.num_channels);
    for (int i = 0; i < header.num_channels; i++) {
      header.pixel_types[i] = TINYEXR_PIXELTYPE_FLOAT; // pixel type of input image
      header.requested_pixel_types[i] = TINYEXR_PIXELTYPE_HALF; // pixel type of output image to be stored in .EXR
    }

    const char* err = NULL; // or nullptr in C++11 or later.
    int ret = SaveEXRImageToFile(&image, &header, outfilename, &err);
    if (ret != TINYEXR_SUCCESS) {
      fprintf(stderr, "Save EXR err: %s\n", err);
      FreeEXRErrorMessage(err); // free's buffer for an error message
      return ret;
    }
    printf("Saved exr file. [ %s ] \n", outfilename);

    free(rgb);

    free(header.channels);
    free(header.pixel_types);
    free(header.requested_pixel_types);

  }
```


Reading deep image EXR file.
See `example/deepview` for actual usage.

```cpp
  const char* input = "deepimage.exr";
  const char* err = NULL; // or nullptr
  DeepImage deepImage;

  int ret = LoadDeepEXR(&deepImage, input, &err);

  // access to each sample in the deep pixel.
  for (int y = 0; y < deepImage.height; y++) {
    int sampleNum = deepImage.offset_table[y][deepImage.width-1];
    for (int x = 0; x < deepImage.width-1; x++) {
      int s_start = deepImage.offset_table[y][x];
      int s_end   = deepImage.offset_table[y][x+1];
      if (s_start >= sampleNum) {
        continue;
      }
      s_end = (s_end < sampleNum) ? s_end : sampleNum;
      for (int s = s_start; s < s_end; s++) {
        float val = deepImage.image[depthChan][y][s];
        ...
      }
    }
  }

```

### deepview

`examples/deepview` is simple deep image viewer in OpenGL.

![DeepViewExample](https://github.com/syoyo/tinyexr/blob/master/examples/deepview/deepview_screencast.gif?raw=true)

## License

3-clause BSD

`tinyexr` tools uses stb, which is licensed under public domain: https://github.com/nothings/stb

`tinyexr` uses some code from OpenEXR, which is licensed under 3-clause BSD license.

## Author(s)

Syoyo Fujita (syoyo@lighttransport.com)

## Contributor(s)

* Matt Ebb (http://mattebb.com): deep image example. Thanks!
* Matt Pharr (http://pharr.org/matt/): Testing tinyexr with OpenEXR(IlmImf). Thanks!
* Andrew Bell (https://github.com/andrewfb) & Richard Eakin (https://github.com/richardeakin): Improving TinyEXR API. Thanks!
* Mike Wong (https://github.com/mwkm): ZIPS compression support in loading. Thanks!
