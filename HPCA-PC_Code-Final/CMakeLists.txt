cmake_minimum_required(VERSION 3.16)
project(nbody_test_registry LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Build Type Default auf Release setzen, falls nicht angegeben
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

set(CMAKE_CXX_FLAGS_RELEASE "-O3" CACHE STRING "Release flags" FORCE)
set(CMAKE_CXX_FLAGS_DEBUG "-O0 -g" CACHE STRING "Debug flags" FORCE)
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -march=native" CACHE STRING "Release flags" FORCE)

# --- Sources ---
add_executable(nbody
  nbody.cpp
  src/backend_registry.cpp
  src/benchmark.cpp
  src/nbody_helper.cpp
  src/body_soa.cpp
  src/backends/sequential.cpp
  src/backends/omp.cpp
  src/backends/simd.cpp
  src/backends/simd_omp.cpp
  src/backends/opencl.cpp
)

# --- Dependencies ---
find_package(SFML 2.5 COMPONENTS graphics window system REQUIRED)
find_package(OpenMP REQUIRED)
target_include_directories(nbody PRIVATE "${CMAKE_SOURCE_DIR}/include")
target_link_libraries(nbody PRIVATE sfml-graphics sfml-window sfml-system OpenMP::OpenMP_CXX)

# Use a local font, or Ubuntu's packaged Open Sans font.
find_file(NBODY_FONT OpenSans-Bold.ttf
  PATHS "${CMAKE_SOURCE_DIR}" /usr/share/fonts/truetype/open-sans)
if(NOT NBODY_FONT)
  message(FATAL_ERROR "Install fonts-open-sans or set -DNBODY_FONT=/path/to/OpenSans-Bold.ttf")
endif()
configure_file("${NBODY_FONT}" "${CMAKE_CURRENT_BINARY_DIR}/OpenSans-Bold.ttf" COPYONLY)

# Small builds for execution checks; normal defaults stay unchanged.
set(NBODY_COUNT 5000 CACHE STRING "Number of orbiting bodies")
set(NBODY_STEPS 1000 CACHE STRING "Benchmark steps")
if(NOT NBODY_COUNT MATCHES "^[1-9][0-9]*$" OR NOT NBODY_STEPS MATCHES "^[1-9][0-9]*$")
  message(FATAL_ERROR "NBODY_COUNT and NBODY_STEPS must be positive integers")
endif()
target_compile_definitions(nbody PRIVATE NBODY_COUNT=${NBODY_COUNT} NBODY_STEPS=${NBODY_STEPS})

# --- Compiler Options ---
target_compile_options(nbody PRIVATE -fno-tree-vectorize)

# --- OpenCL (KORRIGIERT) ---
find_package(OpenCL REQUIRED)

# Header Pfade hinzufügen (manchmal nötig, auch wenn OpenCL::OpenCL das meistens macht)
target_include_directories(nbody PRIVATE ${OpenCL_INCLUDE_DIRS})

# Verlinken mit dem modernen CMake Target
target_link_libraries(nbody PRIVATE OpenCL::OpenCL)

# --- OpenCL Kernel Copy ---
add_custom_command(
  TARGET nbody
  POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E copy_if_different
          "${CMAKE_SOURCE_DIR}/opencl/nbody.cl"
          "$<TARGET_FILE_DIR:nbody>/nbody.cl"
)