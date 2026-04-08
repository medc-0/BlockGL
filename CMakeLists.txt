if(NOT DEFINED CMAKE_CXX_COMPILER)
    find_program(CLANG_CXX_PATH clang++)
    if(CLANG_CXX_PATH)
        set(CMAKE_CXX_COMPILER "${CLANG_CXX_PATH}")
    else()
        find_program(GCC_CXX_PATH g++)
        if(GCC_CXX_PATH)
            set(CMAKE_CXX_COMPILER "${GCC_CXX_PATH}")
        endif()
    endif()
endif()

cmake_minimum_required(VERSION 3.22.1)
project(BlockGL LANGUAGES CXX C)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

file(GLOB_RECURSE BlockGLSources "${CMAKE_SOURCE_DIR}/src/*.cpp")

add_executable(BlockGL ${BlockGLSources})
target_precompile_headers(BlockGL PRIVATE src/BlockGL.h)

add_subdirectory(include/glfw EXCLUDE_FROM_ALL)
add_subdirectory(include/GLAD EXCLUDE_FROM_ALL)
add_subdirectory(include/glm EXCLUDE_FROM_ALL)

add_library(lodepng EXCLUDE_FROM_ALL include/lodepng/lodepng.h include/lodepng/lodepng.cpp)
target_include_directories(lodepng PUBLIC include/lodepng)

target_include_directories(BlockGL PRIVATE 
    ${CMAKE_SOURCE_DIR}/include/FastNoiseLite
    ${CMAKE_SOURCE_DIR}/include/FrustumCulling
)

find_package(Threads REQUIRED)

target_link_libraries(BlockGL PRIVATE 
    glfw 
    glm 
    glad 
    lodepng 
    Threads::Threads
)

include(CheckCXXCompilerFlag)

check_cxx_compiler_flag("-fuse-ld=lld" HAS_LLD)
if(HAS_LLD)
    target_link_options(BlockGL PRIVATE "-fuse-ld=lld")
endif()

if(WIN32 AND (CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU"))
    target_link_options(BlockGL PRIVATE 
        $<$<CONFIG:Release>:-static;-static-libgcc;-static-libstdc++;-Wl,-Bstatic;-lstdc++;-lpthread;-s>
    )
endif()

target_compile_options(BlockGL PRIVATE
    $<$<CONFIG:Release>:-O3 -ffast-math -fno-finite-math-only -ftree-vectorize -mtune=goldmont-plus>
    $<$<CONFIG:Trace>:-O3>
    $<$<CONFIG:Debug>:-g -O0>
)

if (CMAKE_BUILD_TYPE MATCHES Trace)
    target_compile_definitions(BlockGL PRIVATE ENABLE_TRACING)
endif()

include(CheckIPOSupported)
check_ipo_supported(RESULT lto_supported)
if(lto_supported)
    set_property(TARGET BlockGL PROPERTY INTERPROCEDURAL_OPTIMIZATION TRUE)
endif()

add_custom_command(
    TARGET BlockGL POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
    ${CMAKE_SOURCE_DIR}/assets ${CMAKE_CURRENT_BINARY_DIR}/assets
)