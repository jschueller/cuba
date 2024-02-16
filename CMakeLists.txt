cmake_minimum_required (VERSION 3.13)

set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type")

project (cuba C)

option (BUILD_SHARED_LIBS "shared/static" ON)
option (ENABLE_FORK "sampling parallelization using fork syscall" ON)

include (GNUInstallDirs)

include (CheckIncludeFile)
check_include_file(alloca.h HAVE_ALLOCA_H)

include(CheckTypeSize)
check_type_size("long double" HAVE_LONG_DOUBLE)

include(CheckFunctionExists)
if (ENABLE_FORK)
  check_function_exists(fork HAVE_FORK)
endif ()
check_function_exists (shmget HAVE_SHMGET)
if (UNIX)
  find_library(MATH_LIBRARY NAMES m)
endif ()

if (MATH_LIBRARY)
  set(CMAKE_REQUIRED_LIBRARIES ${MATH_LIBRARY})
endif ()
check_function_exists(erf HAVE_ERF)
check_function_exists(powl HAVE_POWL)

if (MSVC)
  set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /wd4267 /wd4996")
endif ()

configure_file(config.h.cmake.in config.h)

add_library(cuba src/common/Fork.c src/common/Global.c src/common/Data.c)
target_include_directories(cuba PRIVATE src/common ${CMAKE_CURRENT_BINARY_DIR})
target_compile_definitions(cuba PRIVATE HAVE_CONFIG_H NOUNDERSCORE REALSIZE=8)
if (MATH_LIBRARY)
  target_link_libraries(cuba PRIVATE ${MATH_LIBRARY})
endif ()

set_property(TARGET cuba PROPERTY POSITION_INDEPENDENT_CODE ON)
set_property(TARGET cuba PROPERTY WINDOWS_EXPORT_ALL_SYMBOLS ON)
set_property(TARGET cuba PROPERTY SOVERSION 0)

set(VARIANTS Vegas Suave Cuhre Divonne)
foreach(VARIANT ${VARIANTS})
  string(TOLOWER "${VARIANT}" variant)
  add_library(cuba_${variant} OBJECT src/${variant}/${VARIANT}.c)
  target_include_directories(cuba_${variant} PRIVATE src/common src/${variant} ${CMAKE_CURRENT_BINARY_DIR})
  set_property(TARGET cuba_${variant} PROPERTY POSITION_INDEPENDENT_CODE ON)
  target_compile_definitions(cuba_${variant} PRIVATE HAVE_CONFIG_H NOUNDERSCORE REALSIZE=8)
  if (WIN32)
    target_compile_definitions(cuba_${variant} PRIVATE _CRT_SECURE_NO_WARNINGS)
  endif ()
  target_link_libraries(cuba PRIVATE cuba_${variant})
endforeach()

install(TARGETS cuba LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR} RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
install(FILES cuba.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

add_executable(demo_c demo/demo-c.c)
target_link_libraries(demo_c PRIVATE cuba)
if (UNIX)
  target_link_libraries(demo_c PRIVATE m)
endif ()
target_include_directories(demo_c PRIVATE .)

enable_testing()
add_test(NAME demo_c COMMAND demo_c)
