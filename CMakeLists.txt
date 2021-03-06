cmake_minimum_required(VERSION 3.11)

project(CACTUS
        VERSION 0.0
        DESCRIPTION "CACTUS"
        HOMEPAGE_URL http://www.github.com/SNL-WaterPower/CACTUS
        LANGUAGES Fortran)

# set compiler-specific options
if(CMAKE_Fortran_COMPILER_ID MATCHES "GNU")
    set(dialect "-O2 -fdefault-real-8 -ffree-line-length-0 -cpp")
    set(bounds "-fbounds-check")
endif()
if(CMAKE_Fortran_COMPILER_ID MATCHES "Intel")
    set(dialect "-O2 -r8 -mkl -fpp")
    set(bounds "-check bounds")
    set(BLA_VENDOR Intel10_64lp)  # force use of MKL BLAS
endif()
if(CMAKE_Fortran_COMPILER_ID MATCHES "PGI")
    set(dialect "-O -r8 -Mpreprocess -Mvect=sse -Mcache_align -Mpre -Mnoframe -Mlre -Mflushz -Mautoinline")
    set(bounds "-C -Mbounds")
endif()

# find LAPACK, OpenMP
enable_language(C)  # For compatibility, some systems complain about this
find_package(LAPACK)

option(OPENMP "Compile with OpenMP enabled?" ON)
if(OPENMP MATCHES ON)
    find_package(OpenMP)
    if (NOT OpenMP_Fortran_FOUND)
        message(FATAL_ERROR "OPENMP option was set to true (default), but no OpenMP was found. Install OpenMP, or try again with -DOPENMP=OFF. Exiting.")
    endif()
    string(APPEND dialect " ${OpenMP_Fortran_FLAGS}")
endif()

list(APPEND CMAKE_Fortran_FLAGS_DEBUG ${bounds})
list(APPEND CMAKE_Fortran_FLAGS ${dialect})

# add VERSION variable
# see https://cmake.org/pipermail/cmake/2018-October/068389.html
find_package(Git QUIET)

if(GIT_EXECUTABLE)
execute_process(
    COMMAND "${GIT_EXECUTABLE}" describe --abbrev=4 --dirty --always --tags
    WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
    RESULT_VARIABLE res
    OUTPUT_VARIABLE CACTUS_VERSION
    # ERROR_QUIET
    OUTPUT_STRIP_TRAILING_WHITESPACE)

    set_property(GLOBAL APPEND
        PROPERTY CMAKE_CONFIGURE_DEPENDS
        "${CMAKE_SOURCE_DIR}/.git/index")
else(NOT GIT_EXECUTABLE)
    # no git found
    set(CACTUS_VERSION "no-git")
endif()

# get appropriate path separator
if(MINGW OR WIN32 OR MSYS)
  set(SEP "\\")
else() # e.g. Linux
  set(SEP "/")
endif()

# add definitions
add_definitions(-DVERSION="${CACTUS_VERSION}")
add_definitions(-DPATHSEP="${SEP}")

# create source lists
file(GLOB_RECURSE mod_sources ${PROJECT_SOURCE_DIR}/mod/**.f95)
file(GLOB_RECURSE prog_sources ${PROJECT_SOURCE_DIR}/src/*.f95)

add_executable(cactus ${prog_sources} ${mod_sources})
target_compile_options(cactus PUBLIC ${LAPACK_LINKER_FLAGS})
if(MINGW OR WIN32 OR MSYS)
    target_link_libraries(cactus ${LAPACK_LIBRARIES} -static)
else()
    target_link_libraries(cactus ${LAPACK_LIBRARIES})
endif()

# build to ./bin
set_target_properties(cactus
    PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/bin/")
