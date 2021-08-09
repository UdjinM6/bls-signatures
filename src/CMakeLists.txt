CMAKE_MINIMUM_REQUIRED(VERSION 3.14 FATAL_ERROR)
set (CMAKE_CXX_STANDARD 17)

if (DEFINED ENV{RELIC_MAIN})
  set(RELIC_GIT_TAG "origin/main")
else ()
  set(RELIC_GIT_TAG "b7b2266a0e4ee6f628f61d3ab638f524a18b52f1")
endif ()

message(STATUS "Relic will be built from: ${RELIC_GIT_TAG}")

FetchContent_Declare(
  relic
  GIT_REPOSITORY https://github.com/relic-toolkit/relic.git
  GIT_TAG        ${RELIC_GIT_TAG}
)
FetchContent_MakeAvailable(relic)

file(GLOB HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/*.hpp)
source_group("SrcHeaders" FILES ${HEADERS})

include_directories(
  ${INCLUDE_DIRECTORIES}
  ${relic_SOURCE_DIR}/include
  ${relic_BINARY_DIR}/include
  )

if (GMP_FOUND)
  include_directories(${GMP_INCLUDES})
endif(GMP_FOUND)

set(C_LIB ${CMAKE_BINARY_DIR}/libbls-dash.a)

add_library(bls-dash ${CMAKE_CURRENT_SOURCE_DIR}/privatekey.cpp)

add_library(blstmp ${HEADERS}
  ${CMAKE_CURRENT_SOURCE_DIR}/privatekey.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/bls.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/chaincode.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/elements.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/extendedprivatekey.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/extendedpublickey.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/legacy.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/schemes.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/threshold.cpp
)

target_compile_definitions(blstmp PRIVATE BLSALLOC_SODIUM=1)
target_link_libraries(blstmp PUBLIC sodium)

if(MSVC)
  add_custom_command(
    OUTPUT ${C_LIB}
    COMMAND ${CMAKE_AR} /OUT:${C_LIB} $<TARGET_FILE:blstmp> $<TARGET_FILE:relic_s>
    DEPENDS blstmp relic_s
  )
  add_custom_target(combined_custom
    DEPENDS ${C_LIB}
  )
else()
  set(OPREFIX object_)
  set(LIBRARIES_TO_COMBINE
    COMMAND ${CMAKE_COMMAND} -E make_directory ${OPREFIX}$<TARGET_NAME:blstmp> && cd ${OPREFIX}$<TARGET_NAME:blstmp> && ${CMAKE_AR} -x $<TARGET_FILE:blstmp>
    COMMAND ${CMAKE_COMMAND} -E make_directory ${OPREFIX}$<TARGET_NAME:relic_s> && cd ${OPREFIX}$<TARGET_NAME:relic_s> && ${CMAKE_AR} -x $<TARGET_FILE:relic_s>
  )

  find_library(GMP_NAME NAMES libgmp.a)
  if(NOT ${GMP_NAME} STREQUAL "GMP_NAME-NOTFOUND")
    list(APPEND LIBRARIES_TO_COMBINE COMMAND ${CMAKE_COMMAND} -E make_directory ${OPREFIX}gmp && cd ${OPREFIX}gmp && ${CMAKE_AR} -x ${GMP_NAME})
  endif()

  add_custom_target(combined_custom
    ${LIBRARIES_TO_COMBINE}
    COMMAND ${CMAKE_AR} -rs ${C_LIB} ${OPREFIX}*/*${CMAKE_C_OUTPUT_EXTENSION}
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    DEPENDS blstmp relic_s
  )
endif()

add_library(combined STATIC IMPORTED GLOBAL)
add_dependencies(combined combined_custom)
target_link_libraries(bls-dash combined)

set_target_properties(combined
  PROPERTIES
  IMPORTED_LOCATION ${C_LIB}
)

file(GLOB includes "${CMAKE_CURRENT_SOURCE_DIR}/*.hpp")
install(DIRECTORY ${relic_SOURCE_DIR}/include/ DESTINATION include/bls-dash)
install(DIRECTORY ${relic_BINARY_DIR}/include/ DESTINATION include/bls-dash)
install(FILES ${includes} DESTINATION include/bls-dash)
install(FILES ${C_LIB} DESTINATION lib)

if(BUILD_BLS_TESTS)
  add_executable(runtest test.cpp)
  pkg_check_modules(CATCH2 catch2)
  if(CATCH2_FOUND)
    # Adding "catch2" subdir to include dirs because "catch.hpp" is included
    # instead of "catch2/catch.hpp"
    if (NOT CATCH2_INCLUDE_DIRS)
      set(CATCH2_INCLUDE_DIRS ${CMAKE_CXX_IMPLICIT_INCLUDE_DIRECTORIES})
    endif()
    list(TRANSFORM CATCH2_INCLUDE_DIRS APPEND /catch2)
    target_include_directories(runtest PRIVATE ${CATCH2_INCLUDE_DIRS})
  else()
    target_include_directories(runtest PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/../contrib/catch)
  endif()
  target_link_libraries(runtest blstmp relic_s)
endif()

if(BUILD_BLS_BENCHMARKS)
  add_executable(runbench test-bench.cpp)
  target_link_libraries(runbench blstmp relic_s)
endif()
