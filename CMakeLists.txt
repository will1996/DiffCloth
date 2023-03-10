cmake_minimum_required(VERSION 3.15)
project(DiffCloth)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()
message(STATUS "set env var")

# Properly setting environment variables for Apple Clang
if (APPLE)
    include_directories("/usr/local/include" "/usr/local/opt/llvm/include")
    link_directories("/usr/local/lib" "/usr/local/opt/llvm/lib")

    set(ENV{CC} "/usr/local/opt/llvm/bin/clang")
    set(ENV{CXX} "/usr/local/opt/llvm/bin/clang-omp++")
    set(ENV{LDFLAGS} "-L/usr/local/opt/libomp/lib")
    set(ENV{CPPFLAGS} "-I/usr/local/opt/libomp/include")

    set(OpenMP_omp_LIBRARY "/usr/local/opt/libomp/lib/libomp.dylib")

    if(CMAKE_C_COMPILER_ID MATCHES "Clang")
        set(OpenMP_C "${CMAKE_C_COMPILER}" CACHE STRING "" FORCE)
        set(OpenMP_C_FLAGS "-Xpreprocessor  -fopenmp -lomp -Wno-unused-command-line-argument" CACHE STRING "" FORCE)

    endif()
    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        set(OpenMP_CXX "${CMAKE_CXX_COMPILER}" CACHE STRING "" FORCE)
        set(OpenMP_CXX_FLAGS "-Xpreprocessor -fopenmp -lomp -Wno-unused-command-line-argument -I/usr/local/opt/libomp/include" CACHE STRING "" FORCE)

    endif()

endif()

set(CMAKE_CXX_FLAGS_RELEASE "-O3")
set(CMAKE_CXX_STANDARD 17)
set(EXECUTABLE_NAME DiffCloth)
set(EXTERNAL_ROOT ${PROJECT_SOURCE_DIR}/external)
set(LIB dependencies/lib)


find_package(OpenMP REQUIRED)
if (OPENMP_FOUND)
    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()

#if (APPLE)
#    add_compile_options(-DUSING_MACBOOK)
#endif ()

if (USE_DEBUG)
    add_compile_options(-DUSE_DEBUG)
endif ()


message("cxx Flags: " ${CMAKE_CXX_FLAGS})

source_group("Shaders" FILES src/gpu/shaders)
source_group("Assets" FILES src/assets)


add_subdirectory(${EXTERNAL_ROOT})


add_definitions(-DSOURCE_PATH=\"${CMAKE_CURRENT_SOURCE_DIR}\")

###############
get_directory_property(EXTERNAL_HEADER
        DIRECTORY ${EXTERNAL_ROOT}
        DEFINITION EXTERNAL_HEADER)


#include_directories(${NANOGUI_EXTRA_INCS})
include_directories(dependencies/include)
include_directories(src/assets)
include_directories(src/gpu)
include_directories(external/LBFGSpp/include)

include_directories(dependencies/lib)


link_directories(${LIB})

# OmegaEngine cpp
file(GLOB HEADERS "src/code/*.h" "src/code/engine/*.h" "src/code/simulation/*.h" "src/code/supports/*.h" "src/code/optimization/*.h")
file(GLOB SRCFILES "src/code/main.cpp" "src/code/*.h" "src/code/*.c" "src/code/engine/*.cpp" "src/code/engine/*.h" "src/code/simulation/*.cpp" "src/code/simulation/*.h" "src/code/supports/*.cpp" "src/code/supports/*.h" "src/code/optimization/*.cpp" "src/code/optimization/*.h")
source_group("Sources" FILES ${SRCFILES})
add_executable(${EXECUTABLE_NAME} ${SRCFILES}  )


target_link_libraries(${EXECUTABLE_NAME} OpenMP::OpenMP_CXX)
target_include_directories(${EXECUTABLE_NAME} PRIVATE ${EXTERNAL_HEADER})

# python binding
if (APPLE)
    set(PYTHON_INCLUDE_DIR "$ENV{HOME}/opt/anaconda3/include/python3.8")
    set(PYTHON_LIBRARIES "$ENV{HOME}/opt/anaconda3/lib/libpython3.8.dylib")
endif ()

#find_package(PythonLibs 3.8 REQUIRED )
find_package(Python 3.8 COMPONENTS Interpreter Development)
#message(${PYTHON_INCLUDE_DIRS})
include_directories(${PYTHON_INCLUDE_DIRS})
include_directories(external/pybind11/include)
add_subdirectory(external/pybind11)


pybind11_add_module(diffcloth_py "src/code/python_interface.cpp" ${HEADERS} ${SRCFILES})
if (APPLE)
  target_link_libraries(diffcloth_py PRIVATE "-undefined dynamic_lookup")
endif ()
# target_link_libraries(diffcloth_py PRIVATE ${PYTHON_LIBRARIES})
target_link_libraries(diffcloth_py PRIVATE pybind11::module)
target_link_libraries(diffcloth_py  PRIVATE OpenMP::OpenMP_CXX)
target_include_directories(diffcloth_py PRIVATE ${EXTERNAL_HEADER})




