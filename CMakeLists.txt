cmake_minimum_required(VERSION 3.16.9)
cmake_policy(SET CMP0074 NEW)

project(PISA CXX C)

if(NOT CMAKE_CXX_STANDARD)
    set(CMAKE_CXX_STANDARD 20)
endif()
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_compile_definitions(BOOST_NO_CXX98_FUNCTION_BASE=1)

option(PISA_BUILD_TOOLS "Build command line tools." ON)
option(PISA_ENABLE_TESTING "Enable testing of the library." ON)
option(PISA_ENABLE_BENCHMARKING "Enable benchmarking of the library." ON)
option(PISA_ENABLE_CLANG_TIDY "Enable static analysis with clang-tidy" OFF)
option(PISA_CLANG_TIDY_EXECUTABLE "clang-tidy executable path" "clang-tidy")
option(PISA_USE_PIC "Enable Position-Independent code globally" ON)
option(PISA_CI_BUILD "Remove debug information from Debug build" OFF)
option(PISA_SANITIZERS "Compile with address and UB sanitizers" OFF)

option(PISA_SYSTEM_GOOGLE_BENCHMARK "Use system installation of Google benchmark library" OFF)
option(PISA_SYSTEM_ONETBB "Use system installation of oneTBB" OFF)
option(PISA_SYSTEM_BOOST "Use system installation of Boost" OFF)
option(PISA_SYSTEM_CLI11 "Use system headers for CLI11" OFF)

if(PISA_USE_PIC)
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
    set(GUMBO_CFLAGS "-fPIC")
endif()

if(PISA_CI_BUILD)
    set(CMAKE_C_FLAGS_DEBUG "")
    set(CMAKE_CXX_FLAGS_DEBUG "")
endif()

configure_file(
  ${PISA_SOURCE_DIR}/include/pisa/pisa_config.hpp.in
  ${PISA_SOURCE_DIR}/include/pisa/pisa_config.hpp
  ESCAPE_QUOTES)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release")
endif()
MESSAGE( STATUS "CMAKE_BUILD_TYPE: " ${CMAKE_BUILD_TYPE} )

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

add_subdirectory(external)

include(ExternalProject)
ExternalProject_Add(gumbo-external
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/gumbo-parser
    BINARY_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/gumbo-parser
    CONFIGURE_COMMAND ./autogen.sh && ./configure --prefix=${CMAKE_BINARY_DIR}/gumbo-parser CFLAGS=${GUMBO_CFLAGS} CC=${CMAKE_C_COMPILER} CXX=${CMAKE_CXX_COMPILER}
	BUILD_BYPRODUCTS ${CMAKE_BINARY_DIR}/gumbo-parser/lib/libgumbo.a
    BUILD_COMMAND ${MAKE})
add_library(gumbo::gumbo STATIC IMPORTED)
set_property(TARGET gumbo::gumbo APPEND PROPERTY INTERFACE_INCLUDE_DIRECTORIES
    ${CMAKE_CURRENT_SOURCE_DIR}/external/gumbo-parser/src)
set_property(TARGET gumbo::gumbo APPEND PROPERTY INCLUDE_DIRECTORIES
    ${CMAKE_CURRENT_SOURCE_DIR}/external/gumbo-parser/src)
set_property(TARGET gumbo::gumbo APPEND PROPERTY IMPORTED_LOCATION
    ${CMAKE_BINARY_DIR}/gumbo-parser/lib/libgumbo.a)
add_dependencies( gumbo::gumbo gumbo-external )

if (UNIX)
   # For hardware popcount and other special instructions
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native")

   # Extensive warnings
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Wno-missing-braces")

   if (PISA_SANITIZERS)
     set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=undefined -fsanitize=address -fno-omit-frame-pointer")
   endif ()

   if(NOT PISA_CI_BUILD)
     set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ggdb") # Add debug info anyway
   endif()

   # modify RPATH when installing to a non-system directory (e.g. /usr/local).
   # required to find libtbb with non-system libtbb.
   list(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/lib" isSystemDir)
   if(NOT PISA_SYSTEM_ONETBB AND "${isSystemDir}" STREQUAL "-1")
      set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
   endif()
endif()

find_package(OpenMP)
if(OPENMP_FOUND)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()

set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)

if (PISA_SYSTEM_BOOST)
    find_package(Boost REQUIRED)
endif()

if (PISA_SYSTEM_ONETBB)
    find_package(TBB REQUIRED)
endif()

file(GLOB_RECURSE PISA_SRC_FILES FOLLOW_SYMLINKS "src/*cpp")
list(SORT PISA_SRC_FILES)

if (PISA_ENABLE_CLANG_TIDY)
    find_program(CLANGTIDY "${PISA_CLANG_TIDY_EXECUTABLE}")
    if(CLANGTIDY)
        set(CMAKE_CXX_CLANG_TIDY ${CLANGTIDY})
    else()
        message(SEND_ERROR "${PISA_CLANG_TIDY_EXECUTABLE} requested but executable not found")
    endif()
endif()

include_directories(include)
add_library(pisa ${PISA_SRC_FILES})
target_include_directories(pisa PUBLIC
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include/pisa>
)
target_link_libraries(pisa
    PUBLIC
        Threads::Threads
        Boost::boost
        mio
        mio_base
        spdlog
        fmt::fmt
        range-v3
        taily
        TBB::tbb
    # These should be made private in the future:
        FastPFor
        streamvbyte
        MaskedVByte
        simdcomp
        QMX
        KrovetzStemmer
        Porter2
    PRIVATE
        gumbo::gumbo
        warcpp
        wapopp
        trecpp
        nlohmann_json::nlohmann_json
)
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    if (CMAKE_CXX_COMPILER_VERSION VERSION_LESS 9.0)
        target_link_libraries(pisa PUBLIC stdc++fs)
    endif()
endif()
target_include_directories(pisa PUBLIC external)

if (PISA_ENABLE_TESTING AND BUILD_TESTING)
    if (ENABLE_COVERAGE)
        # Add code coverage
        list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/external/CMake-codecov/cmake")
        find_package(codecov)
        list(APPEND LCOV_REMOVE_PATTERNS "'${PROJECT_SOURCE_DIR}/external/*'")
    endif()
    enable_testing()
    add_subdirectory(test)
endif()

if (PISA_BUILD_TOOLS)
    if (PISA_SYSTEM_CLI11)
        find_package(CLI11 2 REQUIRED)
    endif()
    add_subdirectory(tools)
endif()

if (PISA_ENABLE_BENCHMARKING)
    add_subdirectory(microbench)
    add_subdirectory(benchmarks)
endif()

if(PISA_COMPILE_HEADERS)
  get_property(PISA_INTERFACE_INCLUDE_DIRECTORIES TARGET pisa PROPERTY INTERFACE_INCLUDE_DIRECTORIES)
  list(APPEND PISA_INCLUDE_DIRECTORIES ${PISA_INTERFACE_INCLUDE_DIRECTORIES})
  get_property(PISA_INTERFACE_LINK_LIBRARIES TARGET pisa PROPERTY INTERFACE_LINK_LIBRARIES)
  foreach(PISA_INTERFACE_LINK_LIBRARY ${PISA_INTERFACE_LINK_LIBRARIES})
    if(NOT PISA_INTERFACE_LINK_LIBRARY MATCHES ".*LINK_ONLY.*")
      get_property(PISA_INTERFACE_LINK_LIBRARY_TYPE TARGET ${PISA_INTERFACE_LINK_LIBRARY} PROPERTY TYPE)
      if(PISA_INTERFACE_LINK_LIBRARY_TYPE STREQUAL "INTERFACE_LIBRARY")
        get_property(PISA_INTERFACE_INCLUDE_DIRECTORIES TARGET ${PISA_INTERFACE_LINK_LIBRARY} PROPERTY INTERFACE_INCLUDE_DIRECTORIES)
        list(APPEND PISA_INCLUDE_DIRECTORIES ${PISA_INTERFACE_INCLUDE_DIRECTORIES})
      else ()
        get_property(PISA_INTERFACE_INCLUDE_DIRECTORIES TARGET ${PISA_INTERFACE_LINK_LIBRARY} PROPERTY INCLUDE_DIRECTORIES)
        list(APPEND PISA_INCLUDE_DIRECTORIES ${PISA_INTERFACE_INCLUDE_DIRECTORIES})
      endif()
    endif()
  endforeach()

  include(CheckCXXSourceCompiles)
  set(CMAKE_REQUIRED_FLAGS "${CMAKE_CXX_FLAGS} -std=c++${CMAKE_CXX_STANDARD} -c")
  get_property(PISA_INTERFACE_INCLUDE_DIRECTORIES TARGET pisa PROPERTY INTERFACE_INCLUDE_DIRECTORIES)
  set(CMAKE_REQUIRED_INCLUDES "${CMAKE_SOURCE_DIR}/external/tbb/include" ${PISA_INCLUDE_DIRECTORIES})

  file(GLOB_RECURSE PISA_HEADER_FILES FOLLOW_SYMLINKS "include/pisa/*.hpp")
  list(SORT PISA_HEADER_FILES)

  foreach(file ${PISA_HEADER_FILES})
    # replace / to _ to fix warnings
    string(REPLACE "/" "_" compilename "${file}")
    string(REPLACE "." "_" compilename "${compilename}")

    if(NOT IsSelfContained${compilename})
      unset(IsSelfContained${compilename} CACHE)
    endif()

    check_cxx_source_compiles(
      "#include \"${file}\"
      int main() { return 0; }" IsSelfContained${compilename})

    if(NOT IsSelfContained${compilename})
      message(FATAL_ERROR
        "Compilation FAILED for ${file}\nCompiler output:\n${OUTPUT}")
    endif()
  endforeach()

endif(PISA_COMPILE_HEADERS)
