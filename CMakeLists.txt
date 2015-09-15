cmake_minimum_required (VERSION 2.6)
project (tcmu-runner)

set (tcmur_VERSION_MAJOR 0)
set (tcmur_VERSION_MINOR 8)
set (tcmur_VERSION_PATCHLEVEL 1)

# configure a header file to pass some of the CMake settings
# to the source code
configure_file (
  "${PROJECT_SOURCE_DIR}/version.h.in"
  "${PROJECT_BINARY_DIR}/version.h"
  )

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Werror -std=c99")

# Stuff for building the main binary
add_executable(tcmu-runner
  main.c
  api.c
  tcmuhandler-generated.c)
target_compile_options(tcmu-runner
  PUBLIC -fPIC -Wl,-E
  )

# Stuff for building the file handler
add_library(handler_file
  SHARED
  file_example.c
  )
set_target_properties(handler_file
  PROPERTIES
  PREFIX ""
  )

# Stuff for building the glfs handler
add_library(handler_glfs
  SHARED
  glfs.c
  )
set_target_properties(handler_glfs
  PROPERTIES
  PREFIX ""
  )

find_library(LIBNL_LIB nl-3)
find_library(LIBNL_GENL_LIB nl-genl-3)
set(LIBNL_LIBS
  ${LIBNL_LIB}
  ${LIBNL_GENL_LIB}
  )

find_path (LIBNL_INCLUDE_DIR
  NAMES
  netlink/netlink.h
  PATH_SUFFIXES
  libnl3
  )

find_package(PkgConfig)
pkg_check_modules(GLIB REQUIRED gio-unix-2.0)

include_directories(
  ${PROJECT_BINARY_DIR}
  ${GLIB_INCLUDE_DIRS}
  ${LIBNL_INCLUDE_DIR}
  )

find_library(PTHREAD pthread)
find_library(DL dl)

find_library(GFAPI gfapi)

target_link_libraries(tcmu-runner
  ${LIBNL_LIB}
  ${LIBNL_GENL_LIB}
  ${GLIB_LIBRARIES}
  ${PTHREAD}
  ${DL}
  )

target_link_libraries(handler_glfs
  ${GFAPI}
  )

add_custom_command(
  OUTPUT tcmuhandler-generated.c tcmuhandler-generated.h
  COMMAND gdbus-codegen tcmu-handler.xml --generate-c-code tcmuhandler-generated --c-generate-object-manager --interface-prefix org.kernel
MAIN_DEPENDENCY tcmu-handler.xml
	)


install(TARGETS tcmu-runner RUNTIME DESTINATION usr/bin)
install(TARGETS handler_file handler_glfs LIBRARY DESTINATION usr/lib/tcmu-runner)
install(FILES tcmu-runner.conf DESTINATION etc/dbus-1/system.d)