cmake_minimum_required(VERSION 3.10)

# Prefer newer policy behavior for IPO/LTO when available
if(POLICY CMP0069)
  cmake_policy(SET CMP0069 NEW)
endif()

project(fmtproject LANGUAGES CXX)

# Default to Release builds for better performance unless explicitly set
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

# C++ standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Options to control optimizations
option(ENABLE_LTO "Enable IPO/LTO if supported" ON)
option(OPTIMIZE_FOR_SIZE "Prefer optimizing for size (-Os) over speed (-O3) in Release" ON)
option(ENABLE_NATIVE_ARCH "Enable -march=native for local builds" OFF)

add_executable(fmtproject main.cpp)

# Try to find fmt, and if not found, fetch it
find_package(fmt CONFIG QUIET)
if(NOT fmt_FOUND)
  include(FetchContent)
  FetchContent_Declare(
    fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG 10.2.1
  )
  FetchContent_MakeAvailable(fmt)
endif()

# Prefer header-only fmt to avoid dynamic link overhead when available
if(TARGET fmt::fmt-header-only)
  target_link_libraries(fmtproject PRIVATE fmt::fmt-header-only)
else()
  target_link_libraries(fmtproject PRIVATE fmt::fmt)
endif()

# Interprocedural optimization (LTO)
if(ENABLE_LTO)
  include(CheckIPOSupported)
  check_ipo_supported(RESULT IPO_SUPPORTED OUTPUT IPO_ERROR)
  if(IPO_SUPPORTED)
    set_property(TARGET fmtproject PROPERTY INTERPROCEDURAL_OPTIMIZATION TRUE)
  else()
    message(STATUS "IPO/LTO not supported: ${IPO_ERROR}")
  endif()
endif()

# Compiler and linker flags tuned for size/startup while keeping performance
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
  # Choose -Os or -O3 for Release
  target_compile_options(fmtproject PRIVATE
    $<$<AND:$<CONFIG:Release>,$<BOOL:${OPTIMIZE_FOR_SIZE}>>:-Os -DNDEBUG>
    $<$<AND:$<CONFIG:Release>,$<NOT:$<BOOL:${OPTIMIZE_FOR_SIZE}>>>:-O3 -DNDEBUG>
    -ffunction-sections -fdata-sections
    $<$<BOOL:${ENABLE_NATIVE_ARCH}>:-march=native>
  )
  target_link_options(fmtproject PRIVATE
    $<$<CONFIG:Release>:-Wl,-O1 -Wl,--gc-sections -Wl,--as-needed -s>
  )
elseif(MSVC)
  target_compile_options(fmtproject PRIVATE
    $<$<CONFIG:Release>:/O2 /DNDEBUG>
  )
  target_link_options(fmtproject PRIVATE
    $<$<CONFIG:Release>:/OPT:REF /OPT:ICF>
  )
endif()