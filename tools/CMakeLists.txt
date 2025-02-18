# Copyright (c) the JPEG XL Project Authors. All rights reserved.
#
# Use of this source code is governed by a BSD-style
# license that can be found in the LICENSE file.

# ICC detection library used by the comparison and viewer tools.
if(JPEGXL_ENABLE_VIEWERS)
if(WIN32)
  find_package(Qt5 QUIET COMPONENTS Widgets)
  if (NOT Qt5_FOUND)
    message(WARNING "Qt5 was not found.")
  else()
    add_library(icc_detect STATIC EXCLUDE_FROM_ALL
      icc_detect/icc_detect_win32.cc
      icc_detect/icc_detect.h
    )
    target_include_directories(icc_detect PRIVATE "${PROJECT_SOURCE_DIR}")
    target_link_libraries(icc_detect PUBLIC Qt5::Widgets)
    if(JPEGXL_DEP_LICENSE_DIR)
      configure_file("${JPEGXL_DEP_LICENSE_DIR}/libqt5widgets5/copyright"
                     ${PROJECT_BINARY_DIR}/LICENSE.libqt5widgets5 COPYONLY)
    endif()  # JPEGXL_DEP_LICENSE_DIR
  endif()
elseif(APPLE)
  find_package(Qt5 QUIET COMPONENTS Widgets)
  if (Qt5_FOUND)
    add_library(icc_detect STATIC EXCLUDE_FROM_ALL
      icc_detect/icc_detect_empty.cc
      icc_detect/icc_detect.h
    )
    target_include_directories(icc_detect PRIVATE "${PROJECT_SOURCE_DIR}")
    target_link_libraries(icc_detect PUBLIC Qt5::Widgets)
  else()
    message(WARNING "APPLE: Qt5 was not found.")
  endif()
else()
  find_package(Qt5 QUIET COMPONENTS Widgets X11Extras)
  find_package(ECM QUIET NO_MODULE)
  if (NOT Qt5_FOUND OR NOT ECM_FOUND)
    if (NOT Qt5_FOUND)
      message(WARNING "Qt5 was not found.")
    else()
      message(WARNING "extra-cmake-modules were not found.")
    endif()
  else()
    set(CMAKE_MODULE_PATH ${ECM_FIND_MODULE_DIR})
    find_package(XCB COMPONENTS XCB)
    if (XCB_FOUND)
      add_library(icc_detect STATIC EXCLUDE_FROM_ALL
        icc_detect/icc_detect_x11.cc
        icc_detect/icc_detect.h
      )
      target_link_libraries(icc_detect PUBLIC jxl-static Qt5::Widgets Qt5::X11Extras XCB::XCB)
    endif ()
  endif()
endif()
endif()  # JPEGXL_ENABLE_VIEWERS

# Tools are added conditionally below.
set(TOOL_BINARIES)

add_library(jxl_tool STATIC EXCLUDE_FROM_ALL
  cmdline.cc
  codec_config.cc
  cpu/cpu.cc
  cpu/os_specific.cc
  tool_version.cc
)
target_compile_options(jxl_tool PUBLIC "${JPEGXL_INTERNAL_FLAGS}")
target_link_libraries(jxl_tool jxl-static)

target_include_directories(jxl_tool
  PUBLIC "${PROJECT_SOURCE_DIR}")

# The JPEGXL_VERSION is set from the builders.
if(NOT DEFINED JPEGXL_VERSION OR JPEGXL_VERSION STREQUAL "")
  find_package(Git QUIET)
  execute_process(
      COMMAND "${GIT_EXECUTABLE}" rev-parse --short HEAD
      OUTPUT_VARIABLE GIT_REV
      WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
      ERROR_QUIET)
  string(STRIP "${GIT_REV}" GIT_REV)
  if(GIT_REV STREQUAL "")
    set(JPEGXL_VERSION "(unknown)")
  endif()
endif()

if(NOT DEFINED JPEGXL_VERSION OR JPEGXL_VERSION STREQUAL "")
  # We are building from a git environment and the user didn't set
  # JPEGXL_VERSION. Make a target that computes the GIT_REV at build-time always
  # but only updates the file if it changed. This allows rebuilds without
  # modifying cmake files to update the JPEGXL_VERSION.
  message(STATUS "Building with JPEGXL_VERSION=${GIT_REV} (auto-updated)")
  add_custom_target(
    tool_version_git
    ${CMAKE_COMMAND}
      -D JPEGXL_ROOT_DIR=${CMAKE_SOURCE_DIR}
      -D DST=${CMAKE_CURRENT_BINARY_DIR}/tool_version_git.h
      -P ${CMAKE_CURRENT_SOURCE_DIR}/git_version.cmake
    BYPRODUCTS "${CMAKE_CURRENT_BINARY_DIR}/tool_version_git.h"
  )
  add_dependencies(jxl_tool tool_version_git)

  set_source_files_properties(tool_version.cc PROPERTIES
    COMPILE_DEFINITIONS JPEGXL_VERSION_FROM_GIT=1)
  target_include_directories(jxl_tool PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
  # Note: Ninja looks for dependencies on the jxl_tool target before running
  # the tool_version_git targets, so when updating the tool_version_git.h the
  # jxl_tool target is not rebuilt. This forces to generate it at configure time
  # if needed.
  execute_process(
    COMMAND ${CMAKE_COMMAND}
      -D JPEGXL_ROOT_DIR=${CMAKE_SOURCE_DIR}
      -D DST=${CMAKE_CURRENT_BINARY_DIR}/tool_version_git.h
      -P ${CMAKE_CURRENT_SOURCE_DIR}/git_version.cmake)
else()
  message(STATUS "Building with JPEGXL_VERSION=${JPEGXL_VERSION}")
  set_source_files_properties(tool_version.cc PROPERTIES
    COMPILE_DEFINITIONS JPEGXL_VERSION=\"${JPEGXL_VERSION}\")
endif()

if(JPEGXL_ENABLE_TOOLS)
  list(APPEND TOOL_BINARIES
    cjxl
    djxl
  )

  # Main compressor.
  add_executable(cjxl
    cjxl.cc
    speed_stats.cc
    cjxl_main.cc
  )
  target_link_libraries(cjxl
      box
      jxl-static
      jxl_extras-static
      jxl_threads-static
  )


  # Main decompressor.
  add_library(djxltool STATIC
    djxl.cc
    speed_stats.cc
  )
  target_link_libraries(djxltool
      box
      jxl-static
      jxl_extras-static
      jxl_threads-static
  )

  add_executable(djxl
      djxl_main.cc
  )
target_link_libraries(djxl djxltool)
endif()  # JPEGXL_ENABLE_TOOLS

# Other developer tools.
if(${JPEGXL_ENABLE_DEVTOOLS})
  list(APPEND TOOL_BINARIES
    fuzzer_corpus
    add_noise
    butteraugli_main
    decode_and_encode
    epf_main
    pq_to_hlg
    render_hlg
    tone_map
    ssimulacra_main
    xyb_range
    jxl_from_tree
  )

  add_executable(fuzzer_corpus fuzzer_corpus.cc)

  add_executable(add_noise add_noise.cc)
  add_executable(ssimulacra_main ssimulacra_main.cc ssimulacra.cc)
  add_executable(butteraugli_main butteraugli_main.cc)
  add_executable(decode_and_encode decode_and_encode.cc)
  add_executable(epf_main epf_main.cc epf.cc epf.h)
  add_executable(pq_to_hlg hdr/pq_to_hlg.cc)
  add_executable(render_hlg hdr/render_hlg.cc)
  add_executable(tone_map hdr/tone_map.cc)
  add_executable(xyb_range xyb_range.cc)
  add_executable(jxl_from_tree jxl_from_tree.cc)
endif()  # JPEGXL_ENABLE_DEVTOOLS

# Benchmark tools.
if(${JPEGXL_ENABLE_BENCHMARK})
  list(APPEND TOOL_BINARIES
    benchmark_xl
  )

  add_executable(benchmark_xl
    benchmark/benchmark_xl.cc
    benchmark/benchmark_args.cc
    benchmark/benchmark_codec.cc
    benchmark/benchmark_file_io.cc
    benchmark/benchmark_stats.cc
    benchmark/benchmark_utils.cc
    benchmark/benchmark_utils.h
    benchmark/benchmark_codec_custom.cc
    benchmark/benchmark_codec_custom.h
    benchmark/benchmark_codec_jxl.cc
    benchmark/benchmark_codec_jxl.h
    benchmark/benchmark_codec_png.cc
    benchmark/benchmark_codec_png.h
    speed_stats.cc
    speed_stats.h
    ../third_party/dirent.cc
  )
  target_link_libraries(benchmark_xl Threads::Threads)
  if(MINGW)
  # MINGW doesn't support glob.h.
  target_compile_definitions(benchmark_xl PRIVATE "-DHAS_GLOB=0")
  endif() # MINGW

  find_package(JPEG)
  if(JPEG_FOUND)
    target_sources(benchmark_xl PRIVATE
      "${CMAKE_CURRENT_LIST_DIR}/benchmark/benchmark_codec_jpeg.cc"
      "${CMAKE_CURRENT_LIST_DIR}/benchmark/benchmark_codec_jpeg.h"
    )
    target_compile_definitions(benchmark_xl PRIVATE -DBENCHMARK_JPEG)
    target_include_directories(benchmark_xl PRIVATE ${JPEG_INCLUDE_DIR})
    target_link_libraries(benchmark_xl ${JPEG_LIBRARIES})
  endif ()

  find_package(PkgConfig)
  pkg_check_modules(WebP IMPORTED_TARGET libwebp)
  if(WebP_FOUND)
    target_sources(benchmark_xl PRIVATE
      "${CMAKE_CURRENT_LIST_DIR}/benchmark/benchmark_codec_webp.cc"
      "${CMAKE_CURRENT_LIST_DIR}/benchmark/benchmark_codec_webp.h"
    )
    target_compile_definitions(benchmark_xl PRIVATE -DBENCHMARK_WEBP)

    # Use the static version of webp if available.
    find_library(WebP_STATIC_LINK_LIBRARY NAMES libwebp.a
        PATHS "${WebP_LIBDIR}")
    if("${WebP_STATIC_LINK_LIBRARY}" STREQUAL "WebP_STATIC_LINK_LIBRARY-NOTFOUND")
      message(WARNING "Using dynamic libwebp")
      target_link_libraries(benchmark_xl PkgConfig::WebP)
    else()
      target_link_libraries(benchmark_xl "${WebP_STATIC_LINK_LIBRARY}")
      target_include_directories(benchmark_xl
          PRIVATE ${WebP_STATIC_INCLUDE_DIRS})
      target_compile_options(benchmark_xl PRIVATE ${WebP_STATIC_CFLAGS_OTHER})
    endif()
  endif()

  pkg_check_modules(AVIF IMPORTED_TARGET libavif)
  if(AVIF_FOUND)
    target_sources(benchmark_xl PRIVATE
      "${CMAKE_CURRENT_LIST_DIR}/benchmark/benchmark_codec_avif.cc"
      "${CMAKE_CURRENT_LIST_DIR}/benchmark/benchmark_codec_avif.h"
    )
    target_compile_definitions(benchmark_xl PRIVATE -DBENCHMARK_AVIF)
    target_link_libraries(benchmark_xl PkgConfig::AVIF)
  endif()
endif()  # JPEGXL_ENABLE_BENCHMARK

# All tool binaries depend on "jxl" library and the tool helpers.
foreach(BINARY IN LISTS TOOL_BINARIES)
  target_include_directories("${BINARY}" PRIVATE "${PROJECT_SOURCE_DIR}")
  target_link_libraries("${BINARY}" box jxl-static jxl_extras-static jxl_threads-static jxl_tool)
endforeach()
install(TARGETS ${TOOL_BINARIES} RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}")
message(STATUS "Building tools: ${TOOL_BINARIES}")

set(FUZZER_BINARIES
  color_encoding_fuzzer
  decode_basic_info_fuzzer
  djxl_fuzzer
  icc_codec_fuzzer
  fields_fuzzer
  rans_fuzzer
  set_from_bytes_fuzzer
  transforms_fuzzer
)

# Fuzzers.
foreach(FUZZER IN LISTS FUZZER_BINARIES)
  if(${JPEGXL_ENABLE_FUZZERS})
    set(BINARY "${FUZZER}")
    add_executable("${BINARY}" "${BINARY}.cc")
    target_link_libraries("${BINARY}" ${JPEGXL_FUZZER_LINK_FLAGS})
  else()
    # When not enabled we want a lightweight alternative for regular fuzzers
    # that just run the target.
    set(BINARY "${FUZZER}_runner")
    add_executable("${BINARY}" EXCLUDE_FROM_ALL
        "fuzzer_stub.cc" "${FUZZER}.cc")
  endif()  # JPEGXL_ENABLE_FUZZERS
  target_include_directories("${BINARY}" PRIVATE "${CMAKE_SOURCE_DIR}")
  if(${FUZZER} STREQUAL djxl_fuzzer)
    target_link_libraries("${BINARY}"
      jxl_dec-static
      jxl_threads-static
    )
  else()
    target_link_libraries("${BINARY}"
      jxl-static
      jxl_extras-static
      jxl_threads-static
      jxl_tool
    )
  endif()
endforeach()

# EMSCRIPTEN doesn't support dynamic libraries so testing for linkage there
# doesn't make much sense.
if(BUILD_TESTING AND TARGET jxl AND NOT JPEGXL_EMSCRIPTEN)
# Library API test. This test is only to check that we can link against the
# shared library from C99 file and don't need to use internal symbols.
file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/tests)
add_executable(libjxl_test libjxl_test.c)
set_property(TARGET libjxl_test PROPERTY C_STANDARD 99)
if(NOT ${SANITIZER} STREQUAL "none")
  # Linking a C test binary with the C++ JPEG XL implementation when using
  # address sanitizer is not well supported by clang 9, so force using clang++
  # for linking this test if a sanitizer is used.
  set_target_properties(libjxl_test PROPERTIES LINKER_LANGUAGE CXX)
endif()  # SANITIZER != "none"
set_target_properties(libjxl_test PROPERTIES PREFIX "tests/")
target_link_libraries(libjxl_test jxl)
if (NOT MSVC)
target_compile_options(libjxl_test PRIVATE -Wall -Wextra -Werror)
if(NOT WIN32)
  target_compile_options(libjxl_test PRIVATE -pedantic)
endif()  # NOT WIN32
endif()  # NOT MSVC

add_test(NAME LibraryCLinkageTest COMMAND libjxl_test)
endif()  # BUILD_TESTING AND TARGET jxl AND NOT JPEGXL_EMSCRIPTEN

# Tools defined in subdirectories.
if(${JPEGXL_ENABLE_VIEWERS})
add_subdirectory(viewer)
add_subdirectory(comparison_viewer)
add_subdirectory(flicker_test)
endif()

add_subdirectory(box)
add_subdirectory(conformance)

if ("${JPEGXL_EMSCRIPTEN}")
# WASM API facade.
add_executable(jxl_emcc jxl_emcc.cc)
target_link_libraries(jxl_emcc jxl-static jxl_extras-static)
set_target_properties(jxl_emcc PROPERTIES LINK_FLAGS "\
  -O3\
  --closure 1 \
  -s ALLOW_MEMORY_GROWTH=1 \
  -flto \
  --llvm-lto 1 \
  -s DISABLE_EXCEPTION_CATCHING=1 \
  -s MODULARIZE=1 \
  -s FILESYSTEM=0 \
  -s EXPORT_NAME=\"JxlCodecModule\"\
  -s \"EXPORTED_FUNCTIONS=[\
    _jxlCompress,\
    _jxlDecompress,\
    _free,\
    _malloc\
  ]\"\
")
endif ()  # JPEGXL_EMSCRIPTEN

if(JPEGXL_ENABLE_JNI)
find_package(JNI QUIET)
find_package(Java QUIET)

if ("${JNI_FOUND}" AND "${Java_FOUND}")
  include(UseJava)

  # decoder_jni_onload.cc might be necessary for Android; not used yet.
  add_library(jxl_jni SHARED jni/org/jpeg/jpegxl/wrapper/decoder_jni.cc)
  target_include_directories(jxl_jni PRIVATE "${JNI_INCLUDE_DIRS}" "${PROJECT_SOURCE_DIR}")
  target_link_libraries(jxl_jni PUBLIC jxl_dec-static jxl_threads-static)

  add_jar(jxl_jni_wrapper
    jni/org/jpeg/jpegxl/wrapper/Decoder.java
    jni/org/jpeg/jpegxl/wrapper/DecoderJni.java
    jni/org/jpeg/jpegxl/wrapper/ImageData.java
    jni/org/jpeg/jpegxl/wrapper/PixelFormat.java
    jni/org/jpeg/jpegxl/wrapper/Status.java
    jni/org/jpeg/jpegxl/wrapper/StreamInfo.java
  )
  get_target_property(JXL_JNI_WRAPPER_JAR jxl_jni_wrapper JAR_FILE)

  add_jar(jxl_jni_wrapper_test
    SOURCES jni/org/jpeg/jpegxl/wrapper/DecoderTest.java
    INCLUDE_JARS jxl_jni_wrapper
  )
  get_target_property(JXL_JNI_WRAPPER_TEST_JAR jxl_jni_wrapper_test JAR_FILE)

  if(NOT "${SANITIZER}" MATCHES ".san")
    # NB: Vanilla OpenJDK 8 / 11 are known to work well (i.e. either
    #     "which java" or JAVA_HOME environment variable point to the path like
    #     "/usr/lib/jvm/java-xx-openjdk-yyy" on Debian Linux).
    add_test(
      NAME test_jxl_jni_wrapper
      COMMAND ${Java_JAVA_EXECUTABLE}
              -cp "${JXL_JNI_WRAPPER_JAR}:${JXL_JNI_WRAPPER_TEST_JAR}"
              -Dorg.jpeg.jpegxl.wrapper.lib=$<TARGET_FILE:jxl_jni>
              org.jpeg.jpegxl.wrapper.DecoderTest
    )
  endif()  # JPEGXL_ENABLE_FUZZERS
endif()  # JNI_FOUND & Java_FOUND
endif()  # JPEGXL_ENABLE_JNI
