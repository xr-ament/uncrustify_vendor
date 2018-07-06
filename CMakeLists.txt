cmake_minimum_required(VERSION 3.5)

project(uncrustify_vendor)

find_package(ament_cmake REQUIRED)

macro(build_uncrustify)
  set(extra_cmake_args)
  if(DEFINED CMAKE_BUILD_TYPE)
    list(APPEND extra_cmake_args -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE})
  endif()

  include(ExternalProject)

  if(WIN32)
    set(patch_exe patch.exe)
  else()
    set(patch_exe patch)
  endif()

  # Get uncrustify 0.67.0
  ExternalProject_Add(uncrustify-0.67
    URL https://github.com/uncrustify/uncrustify/archive/uncrustify-0.67.tar.gz
    URL_MD5 5469a25979c6f943a0178e40dabf1c65
    TIMEOUT 600
    CMAKE_ARGS
      -DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}_install
      ${extra_cmake_args}
      -Wno-dev
  PATCH_COMMAND
    ${patch_exe} -p1 -N < ${CMAKE_CURRENT_SOURCE_DIR}/install-patch.diff
  )

  # The external project will install to the build folder, but we'll install that on make install.
  install(
    DIRECTORY
      ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}_install/
    DESTINATION
      ${CMAKE_INSTALL_PREFIX}
  )
endmacro()

execute_process(
  COMMAND uncrustify --version
  RESULT_VARIABLE res
  OUTPUT_VARIABLE out
  ERROR_VARIABLE err
  OUTPUT_STRIP_TRAILING_WHITESPACE
  ERROR_STRIP_TRAILING_WHITESPACE
)
set(need_local_build FALSE)
if(NOT res EQUAL 0)
  set(need_local_build TRUE)
else()
  # version of uncrustify before 0.65 user a different versionning scheme so the regex wont be matched
  string(REGEX REPLACE "^Uncrustify-(.*)_f$" "\\1" version_prefix_match "${out}")
  if(version_prefix_match STREQUAL "" OR version_prefix_match VERSION_LESS 0.66.1)
    set(need_local_build TRUE)
  endif()
endif()
if(need_local_build)
  build_uncrustify()
endif()
  
# this ensures that the package has an environment hook setting the PATH
ament_package()
