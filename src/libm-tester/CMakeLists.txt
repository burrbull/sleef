# Note: We are assuming SLEEF is the CMake root project.
# TODO: Remove constraint: do not use CMAKE_BINARY_DIR and CMAKE_SOURCE_DIR
link_directories(${CMAKE_BINARY_DIR}/lib)                 # libsleef
include_directories(${CMAKE_BINARY_DIR}/include)          # sleef.h
include_directories(${CMAKE_SOURCE_DIR}/src/libm)         # rename.h
include_directories(${CMAKE_BINARY_DIR}/src/libm/include) # rename headers

set(LIB_SLEEF sleef)
find_library(LIB_MPFR mpfr)
set(EXTRA_LIBS m
  ${LIB_MPFR})

# Compile executable 'iut'
add_executable(${TARGET_IUT} iut.c testerutil.c)
target_link_libraries(${TARGET_IUT} ${LIB_SLEEF} ${EXTRA_LIBS})
set(IUT_LIST ${TARGET_IUT})

# Add vector extension `iut`s
if(COMPILER_SUPPORTS_SSE2)
  add_executable(${TARGET_IUTSSE2} iutsimd.c iutsimdmain.c testerutil.c)
  target_compile_options(${TARGET_IUTSSE2}
    PRIVATE ${FLAGS_ENABLE_SSE2})
  target_compile_definitions(${TARGET_IUTSSE2}
    PRIVATE ENABLE_SSE2=1)
  target_link_libraries(${TARGET_IUTSSE2} ${LIB_SLEEF} ${EXTRA_LIBS})

  list(APPEND IUT_LIST ${TARGET_IUTSSE2})
endif(COMPILER_SUPPORTS_SSE2)

# Compile executable 'tester'
add_executable(${TARGET_TESTER} tester.c testerutil.c)
target_link_libraries(${TARGET_TESTER} ${EXTRA_LIBS} ${LIB_SLEEF})
target_compile_definitions(${TARGET_TESTER}
  PRIVATE USEMPFR=1)
target_compile_options(${TARGET_TESTER} PRIVATE -Wno-unused-result)

# Set C standard requirement (-std=gnu99 for gcc)
set_target_properties(${TARGET_TESTER} ${IUT_LIST} PROPERTIES
  C_STANDARD 99)

# Target 'test-libm' runs the libm tests
add_custom_target(${TARGET_TEST}
  COMMAND ${CMAKE_COMMAND}
    -DTARGET_TESTER="${TARGET_TESTER}"
    -DIUT_LIST="${IUT_LIST}"
    -DLOCATIONS_FILE="${PROJECT_BINARY_DIR}/DefineLocations.cmake"
    -P ${SLEEF_SCRIPT_PATH}/ExecuteTests.cmake
  COMMENT "Running tests..."
  DEPENDS ${TARGET_TESTER} ${IUT_LIST})
# Tests depends on the library
add_dependencies(${TARGET_TEST} ${TARGET_LIBSLEEF})