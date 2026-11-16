# libmain.so: BOTWRecompAndroid runtime exclusivo para Xiaomi 12 Pro (Snapdragon 8 Gen 1 / Adreno 730)
cmake_minimum_required(VERSION 3.22)
project(botwrecompandroid C CXX ASM)

set(R ${CMAKE_CURRENT_SOURCE_DIR}/../../../../../..)   # the repository
get_filename_component(R ${R} ABSOLUTE)
set(DEPS ${R}/android/deps/out)                        # android/deps/build_deps.sh
if(NOT EXISTS ${DEPS}/lib/libSDL3.so)
    message(FATAL_ERROR "run android/deps/build_deps.sh first")
endif()
if(NOT BOTW_GEN_DIR)
    message(FATAL_ERROR "BOTW_GEN_DIR (the Gradle build's generated files: smaa_hlsl.inc) is not set")
endif()

include(${R}/android/gamelib_id.cmake)
message(STATUS "GAMELIB_ID ${GAMELIB_ID} [TARGET: SM8450 / Cortex-X2 / Adreno 730]")

add_subdirectory(${R}/third_party/libadrenotools ${CMAKE_BINARY_DIR}/adrenotools)

add_library(SDL3 SHARED IMPORTED)
set_target_properties(SDL3 PROPERTIES IMPORTED_LOCATION ${DEPS}/lib/libSDL3.so)
add_library(shaderc_shared SHARED IMPORTED)
set_target_properties(shaderc_shared PROPERTIES IMPORTED_LOCATION ${DEPS}/lib/libshaderc_shared.so)

file(GLOB RT_SRC CONFIGURE_DEPENDS ${R}/runtime/*.c ${R}/runtime/hle/*.c ${R}/runtime/services/*.c ${R}/runtime/nvn/*.c
     ${R}/runtime/nvn/maxwell/*.c)

set(HAVE_FFMPEG FALSE)
if(EXISTS ${DEPS}/lib/libavformat.so)
    set(HAVE_FFMPEG TRUE)
    foreach(l avcodec avformat avutil swscale)
        add_library(${l} SHARED IMPORTED)
        set_target_properties(${l} PROPERTIES IMPORTED_LOCATION ${DEPS}/lib/lib${l}.so)
    endforeach()
else()
    list(REMOVE_ITEM RT_SRC ${R}/runtime/hle/movie.c)
endif()

file(GLOB VK_SRC CONFIGURE_DEPENDS ${R}/runtime/nvn/vulkan/*.c ${R}/runtime/nvn/vulkan/*.cpp)

# Lossless Scaling Frame Generation (LSFG) activado para duplicar FPS en Adreno 730
option(BOTW_LSFG "Lossless Scaling frame generation (GPL-3.0-or-later)" ON)
if(BOTW_LSFG)
    file(GLOB LSFG_SRC CONFIGURE_DEPENDS ${R}/runtime/nvn/vulkan/lsfg/*.c)
    list(APPEND VK_SRC ${LSFG_SRC})
else()
    list(APPEND VK_SRC ${R}/runtime/nvn/vulkan/nolsfg/lsfg_stub.c)
endif()

file(GLOB UI_SRC CONFIGURE_DEPENDS ${R}/runtime/ui/*.mm)
set(IMGUI ${R}/third_party/imgui)
set(IMGUI_SRC ${IMGUI}/imgui.cpp ${IMGUI}/imgui_draw.cpp ${IMGUI}/imgui_tables.cpp ${IMGUI}/imgui_widgets.cpp
    ${IMGUI}/backends/imgui_impl_sdl3.cpp ${IMGUI}/backends/imgui_impl_vulkan.cpp)

add_library(main SHARED ${RT_SRC} ${R}/runtime/platform/android.c ${VK_SRC} ${UI_SRC} ${IMGUI_SRC}
            ${DEPS}/volk/volk.c ${R}/android/common/guest_window.c ${R}/runtime/native_gate.S)

set_source_files_properties(${UI_SRC} PROPERTIES LANGUAGE CXX COMPILE_OPTIONS "-xc++")
set_source_files_properties(${R}/runtime/gamelib.c PROPERTIES COMPILE_DEFINITIONS "GAMELIB_ID=\"${GAMELIB_ID}\"")
set_source_files_properties(${R}/runtime/gamebuild.c PROPERTIES COMPILE_DEFINITIONS "GAMELIB_ID=\"${GAMELIB_ID}\"")

add_custom_target(benchmark_commit BYPRODUCTS ${CMAKE_BINARY_DIR}/benchmark/benchmark_commit.h
    COMMAND ${CMAKE_COMMAND} -DR=${R} -DOUT=${CMAKE_BINARY_DIR}/benchmark/benchmark_commit.h -P ${CMAKE_CURRENT_LIST_DIR}/benchmark_commit.cmake VERBATIM)
add_dependencies(main benchmark_commit)
target_include_directories(main PRIVATE ${CMAKE_BINARY_DIR}/benchmark)

target_include_directories(main PRIVATE ${R}/runtime ${R}/runtime/nvn ${IMGUI} ${IMGUI}/backends ${DEPS}/include
                           ${DEPS}/volk ${BOTW_GEN_DIR} ${R}/third_party/smaa/Textures ${R}/android/common)

target_compile_definitions(main PRIVATE
    VK_NO_PROTOTYPES VK_USE_PLATFORM_ANDROID_KHR IMGUI_IMPL_VULKAN_USE_VOLK
    VMA_STATIC_VULKAN_FUNCTIONS=0 VMA_DYNAMIC_VULKAN_FUNCTIONS=1
    TARGET_SOC_SM8450=1
    ADRENO_730_OPT=1)

if(HAVE_FFMPEG)
    target_link_libraries(main PRIVATE avcodec avformat avutil swscale)
else()
    target_compile_definitions(main PRIVATE LARECOMP_NO_FFMPEG)
endif()

# Optimizacion exclusiva ARMv9-A / Cortex-X2 + SVE2 sin simbolos debug (-g eliminado para menor uso de RAM)
target_compile_options(main PRIVATE
    -O3
    -mcpu=cortex-x2+crypto+dotprod+fp16+rcpc+sve2
    -mtune=cortex-x2
    -fno-strict-aliasing
    -Wall
    -Wno-unused-function
    -Wno-format
    $<$<COMPILE_LANGUAGE:CXX>:-std=c++17>)

target_link_libraries(main PRIVATE SDL3 shaderc_shared adrenotools android log dl m)
# Mantener tablas HLE intactas + eliminar codigo muerto
target_link_options(main PRIVATE -Wl,-z,nostart-stop-gc -Wl,--gc-sections)
