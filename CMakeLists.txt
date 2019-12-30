cmake_minimum_required(VERSION 3.0)

set(AACS_VERSION_MAJOR 0)
set(AACS_VERSION_MINOR 9)
set(AACS_VERSION_MICRO 0)

project(libaacs VERSION ${AACS_VERSION_MAJOR}.${AACS_VERSION_MINOR}.${AACS_VERSION_MICRO} LANGUAGES C)

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/src/libaacs/aacs-version.h.in
               ${CMAKE_CURRENT_SOURCE_DIR}/src/libaacs/aacs-version.h)

find_package(libgpg-error 1.2.7 REQUIRED NO_MODULE)
find_package(libgcrypt 1.7 REQUIRED NO_MODULE)
find_package(iconv REQUIRED NO_MODULE)

if(MSVC)
  #set(CMAKE_DEBUG_POSTFIX "d")
endif()

find_program(BISON NAMES bison win_bison)
include(FindPackageHandleStandardArgs)
find_package_handle_standard_args(BISON REQUIRED_VARS BISON)
if(NOT ${BISON_FOUND})
  message(FATAL_ERROR "BISON not found, can't generate error codes")
endif()
find_program(FLEX NAMES flex win_flex)
find_package_handle_standard_args(FLEX REQUIRED_VARS FLEX)
if(NOT ${FLEX_FOUND})
  message(FATAL_ERROR "FLEX not found, can't generate error codes")
endif()

add_library(libaacs
  cmake/libaacs.def
  src/file/dirs.h
  src/file/dirs_win32.c
  src/file/file.c
  src/file/file.h
  src/file/file_win32.c
  src/file/filesystem.c
  src/file/filesystem.h
  src/file/keydb.h
  src/file/keydbcfg-lexer.c
  src/file/keydbcfg-parser.c
  src/file/keydbcfg.c
  src/file/keydbcfg.h
  src/file/mmc_device.h
  src/file/mmc_device_win32.c
  src/file/path.h
  src/libaacs/aacs.c
  src/libaacs/aacs.h
  src/libaacs/cci.c
  src/libaacs/cci.h
  src/libaacs/cci_data.h
  src/libaacs/content_cert.c
  src/libaacs/content_cert.h
  src/libaacs/crypto.c
  src/libaacs/crypto.h
#  src/libaacs/ec.c
  src/libaacs/mkb.c
  src/libaacs/mkb.h
  src/libaacs/mmc.c
  src/libaacs/mmc.h
  src/libaacs/unit_key.c
  src/libaacs/unit_key.h
  src/libaacs/aacs-version.h
  src/util/attributes.h
  src/util/logging.c
  src/util/logging.h
  src/util/macro.h
  src/util/strutl.c
  src/util/strutl.h
)

SET_TARGET_PROPERTIES(libaacs PROPERTIES LINK_FLAGS "/DEF:\"libaacs.def\"")

target_compile_definitions(libaacs
  PRIVATE
  HAVE_CONFIG_H
  PKGDATADIR=""
  _CRT_SECURE_NO_WARNINGS
  _CRT_NONSTDC_NO_WARNINGS
  __PRETTY_FUNCTION__=__FUNCTION__
)

if(WINDOWS_STORE)
  target_compile_definitions(libaacs
    PRIVATE
      MS_APP
  )
else()
  target_link_libraries(libaacs
    PRIVATE
    shell32.lib
  )
endif()

target_link_libraries(libaacs
  PRIVATE
  kernel32.lib
)


target_link_options(libaacs
  PRIVATE
    /debug:full
)
  
target_include_directories(libaacs
  PRIVATE
  $<BUILD_INTERFACE:.;cmake;src;src/util;${CMAKE_CURRENT_BINARY_DIR}>
  INTERFACE
  $<INSTALL_INTERFACE:include/libaacs>
)

add_custom_command(
  OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/src/file/keydbcfg-parser.c ${CMAKE_CURRENT_SOURCE_DIR}/src/file/keydbcfg-lexer.c
  COMMAND ${FLEX} -L ${CMAKE_CURRENT_SOURCE_DIR}/src/file/keydbcfg-lexer.l
  COMMAND ${CMAKE_COMMAND} -E rename ${CMAKE_CURRENT_BINARY_DIR}/lex.yy.c ${CMAKE_CURRENT_SOURCE_DIR}/src/file/keydbcfg-lexer.c
  COMMAND ${BISON} -l -y -d -p libaacs_yy -o ${CMAKE_CURRENT_SOURCE_DIR}/src/file/keydbcfg-parser.c ${CMAKE_CURRENT_SOURCE_DIR}/src/file/keydbcfg-parser.y
)

target_link_libraries(libaacs PRIVATE libgpg-error::libgpg-error libgcrypt::libgcrypt iconv::iconv)

if (NOT WINDOWS_STORE)
  set(EX_SRCS
    ${CMAKE_CURRENT_SOURCE_DIR}/src/examples/aacs_info.c
  )

  ADD_EXECUTABLE(aacsinfo ${EX_SRCS})
  target_compile_definitions(aacsinfo
    PRIVATE
    HAVE_CONFIG_H
    _CRT_SECURE_NO_WARNINGS
    _CRT_NONSTDC_NO_WARNINGS)
    
  target_include_directories(aacsinfo
    PRIVATE
    $<BUILD_INTERFACE:.;cmake;src;${CMAKE_CURRENT_BINARY_DIR}>
    INTERFACE
  )
  add_dependencies(aacsinfo libaacs)
  target_link_libraries(aacsinfo PRIVATE libaacs)
endif(NOT WINDOWS_STORE)

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY AnyNewerVersion
)

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY AnyNewerVersion
)

install(TARGETS ${PROJECT_NAME} EXPORT ${PROJECT_NAME}
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
)

install(FILES src/libaacs/aacs.h src/file/filesystem.h src/libaacs/aacs-version.h DESTINATION include/${PROJECT_NAME})

if(MSVC)
  set_target_properties(libaacs
    PROPERTIES
      COMPILE_PDB_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}
      COMPILE_PDB_NAME libaacs
      COMPILE_PDB_NAME_DEBUG libaacsd
  )
  install(FILES
    ${PROJECT_BINARY_DIR}/RelWithDebInfo/libaacs.pdb
    DESTINATION lib
    CONFIGURATIONS RelWithDebInfo
  )
  install(FILES
    ${PROJECT_BINARY_DIR}/Debug/libaacsd.pdb
    DESTINATION lib
    CONFIGURATIONS Debug
  )
endif()

install(EXPORT ${PROJECT_NAME}
  FILE
    ${PROJECT_NAME}.cmake
  NAMESPACE
    ${PROJECT_NAME}::
  DESTINATION
    lib/cmake/${PROJECT_NAME}
)

install(
  FILES
    cmake/${PROJECT_NAME}-config.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake
  DESTINATION
    lib/cmake/${PROJECT_NAME}
)
