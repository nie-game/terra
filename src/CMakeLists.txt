###
### Terra Generated Source Files
###

add_custom_command(
  OUTPUT "${PROJECT_BINARY_DIR}/internalizedfiles.h"
  DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/geninternalizedfiles.lua"
    "${PROJECT_SOURCE_DIR}/lib/std.t"
    "${PROJECT_SOURCE_DIR}/lib/parsing.t"
    LuaJIT
  COMMAND ${LUAJIT_EXECUTABLE} "${CMAKE_CURRENT_SOURCE_DIR}/geninternalizedfiles.lua" ${PROJECT_BINARY_DIR}/internalizedfiles.h ${CLANG_RESOURCE_DIR} "%.h$" ${CLANG_RESOURCE_DIR} "%.modulemap$" "${PROJECT_SOURCE_DIR}/lib" "%.t$"
  WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
  VERBATIM
)

list(APPEND TERRA_LIB_LUA_SRC
  terralib.lua
  strict.lua
  cudalib.lua
  asdl.lua
  terralist.lua
)

foreach(LUA_SRC ${TERRA_LIB_LUA_SRC})
  get_filename_component(LUA_BASE ${LUA_SRC} NAME_WE)
  set(LUA_BC_GEN "${LUA_BASE}.bc")
  set(LUA_H_GEN "${LUA_BASE}.h")
  list(APPEND TERRA_LIB_LUA_GEN "${PROJECT_BINARY_DIR}/${LUA_H_GEN}")
  add_custom_command(
    OUTPUT "${PROJECT_BINARY_DIR}/${LUA_BC_GEN}"
    DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/${LUA_SRC}" LuaJIT
    COMMAND ${LUAJIT_EXECUTABLE} -bg "${CMAKE_CURRENT_SOURCE_DIR}/${LUA_SRC}" "${PROJECT_BINARY_DIR}/${LUA_BC_GEN}"
    VERBATIM
  )
  add_custom_command(
    OUTPUT "${PROJECT_BINARY_DIR}/${LUA_H_GEN}"
    DEPENDS "${PROJECT_BINARY_DIR}/${LUA_BC_GEN}" LuaJIT
    COMMAND "${LUAJIT_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/genheader.lua" "${PROJECT_BINARY_DIR}/${LUA_BC_GEN}" "${PROJECT_BINARY_DIR}/${LUA_H_GEN}"
    VERBATIM
  )
endforeach()

# This header isn't generated but needs to be relocated to fit the
# build directory structure.
add_custom_command(
  OUTPUT ${PROJECT_BINARY_DIR}/include/terra/terra.h
  DEPENDS
    ${PROJECT_SOURCE_DIR}/release/include/terra/terra.h
  COMMAND "${CMAKE_COMMAND}" -E copy "${PROJECT_SOURCE_DIR}/release/include/terra/terra.h" "${PROJECT_BINARY_DIR}/include/terra/"
  VERBATIM
)

add_custom_target(
  TerraGeneratedFiles
  DEPENDS
    ${PROJECT_BINARY_DIR}/internalizedfiles.h
    ${PROJECT_BINARY_DIR}/include/terra/terra.h
    ${TERRA_LIB_LUA_GEN}
)

###
### Terra Library
###

list(APPEND TERRA_LIB_SRC
  tkind.cpp        tkind.h
  tcompiler.cpp    tcompiler.h
  tcompilerstate.h
  tllvmutil.cpp    tllvmutil.h
  tcwrapper.cpp    tcwrapper.h
  tinline.cpp      tinline.h
  terra.cpp
  lparser.cpp      lparser.h
  lstring.cpp      lstring.h
  lobject.cpp      lobject.h
  lzio.cpp         lzio.h
  llex.cpp         llex.h
  lctype.cpp       lctype.h
  treadnumber.c    treadnumber.h
  tcuda.cpp        tcuda.h
  tdebug.cpp       tdebug.h
  tinternalizedfiles.cpp
  lj_strscan.c     lj_strscan.h

  ${PROJECT_BINARY_DIR}/include/terra/terra.h
)

list(APPEND TERRA_BIN_SRC
  main.cpp
  linenoise.cpp linenoise.h
)

add_library(TerraObjectFiles OBJECT ${TERRA_LIB_SRC})

target_include_directories(TerraObjectFiles
  PRIVATE
    ${PROJECT_BINARY_DIR}
    ${PROJECT_BINARY_DIR}/include/terra
    ${LLVM_INCLUDE_DIRS}
    ${CLANG_INCLUDE_DIRS}
)

if(WIN32)
  target_include_directories(TerraObjectFiles
    PRIVATE
      ${PROJECT_SOURCE_DIR}/msvc
  )
endif()

target_compile_definitions(TerraObjectFiles
  PRIVATE
    LLVM_VERSION=${LLVM_VERSION_MAJOR}${LLVM_VERSION_MINOR}
)

if(WIN32)
  target_compile_options(TerraObjectFiles
    PRIVATE
      # FIXME: Find portable ways to do all these
      /MP8    # parallel build
      /nologo
      /EHsc   # extern "C" functions don't throw exceptions
      /w      # disable warnings
      #/MD    # Don't do this, cmake chooses the correct version automatically
      /Zi     # debug info
  )
  target_compile_definitions(TerraObjectFiles
    PRIVATE
      _CRT_SECURE_NO_DEPRECATE
      NOMINMAX
  )
else()
  target_compile_options(TerraObjectFiles
    PRIVATE
      # FIXME: Find portable ways to do all these
      -fno-common
      -Wcast-qual
      $<$<COMPILE_LANGUAGE:CXX>:-Woverloaded-virtual>
      $<$<COMPILE_LANGUAGE:CXX>:-fvisibility-inlines-hidden>
  )
endif()

target_compile_options(TerraObjectFiles
  PRIVATE
    # LLVM provides these as flags, so we have to put them here.
    ${ALL_LLVM_DEFINITIONS}
)

if(NOT WIN32 AND NOT ${LLVM_ENABLE_RTTI})
  target_compile_options(TerraObjectFiles
    PRIVATE
      # FIXME: Find portable ways to do all these
      $<$<COMPILE_LANGUAGE:CXX>:-fno-rtti>
  )
endif()

if(NOT ${LLVM_ENABLE_ASSERTIONS})
  target_compile_definitions(TerraObjectFiles
    PRIVATE
      TERRA_LLVM_HEADERS_HAVE_NDEBUG
  )
endif()

target_compile_definitions(TerraObjectFiles
  PRIVATE
    ${TERRA_VERSION_DEFINITIONS}
)

set_target_properties(TerraObjectFiles PROPERTIES POSITION_INDEPENDENT_CODE ON)

add_dependencies(TerraObjectFiles TerraGeneratedFiles)

add_library(TerraLibrary
  STATIC
    $<TARGET_OBJECTS:TerraObjectFiles>
    ${ALL_LLVM_OBJECTS}
    ${LUAJIT_OBJECTS}
)

set_source_files_properties(
  ${ALL_LLVM_OBJECTS}
  ${LUAJIT_OBJECTS}
  PROPERTIES
    EXTERNAL_OBJECT true
    GENERATED true)

add_dependencies(TerraLibrary LuaJIT)
add_dependencies(TerraLibrary LLVMObjectFiles)

set_target_properties(TerraLibrary PROPERTIES OUTPUT_NAME terra_s)

if(WIN32)

  list(APPEND TERRA_EXPORTS
    /EXPORT:terra_init
    /EXPORT:terra_initwithoptions
    /EXPORT:terra_load
    /EXPORT:terra_loadfile
    /EXPORT:terra_loadbuffer
    /EXPORT:terra_loadstring
    /EXPORT:terra_llvmshutdown
  )
endif()

###
### Terra Executable
###

list(APPEND TERRA_EXE_SRC
  main.cpp
)

if(WIN32)
  list(APPEND TERRA_EXE_SRC
    ${PROJECT_SOURCE_DIR}/msvc/ext/getopt.c ${PROJECT_SOURCE_DIR}/msvc/ext/getopt.h
    ${PROJECT_SOURCE_DIR}/msvc/ext/getopt_long.c
    ${PROJECT_SOURCE_DIR}/msvc/ext/inttypes.h
    ${PROJECT_SOURCE_DIR}/msvc/ext/setjmp.h
  )
else()
  list(APPEND TERRA_EXE_SRC
    linenoise.cpp linenoise.h
  )
endif()

if(APPLE)
  set_target_properties(TerraExecutable
    PROPERTIES
      # Makes LuaJIT happy (otherwise luaL_newstate returns NULL). Has to be
      # set as a property otherwise target_link_libraries thinks the arguments
      # are libraries.
      LINK_FLAGS "-pagezero_size 10000 -image_base 100000000"
  )
  target_link_libraries(TerraExecutable
    PRIVATE
      -Wl,-force_load,$<TARGET_LINKER_FILE:TerraLibrary>
      ${LUAJIT_LIBRARIES}
      ${ALL_LLVM_LIBRARIES}
      ${LLVM_SYSTEM_LIBRARIES}
  )
  add_dependencies(TerraExecutable TerraLibrary)
elseif(UNIX)
  if(NOT TERRA_STATIC_LINK_LLVM OR NOT TERRA_STATIC_LINK_LUAJIT)
    target_link_libraries(TerraExecutable
      PRIVATE
        "-Wl,-rpath,$ORIGIN/../lib"
    )
  endif()
else()
endif()

if(WIN32)
  target_link_libraries(TerraExecutable
    PRIVATE
      shlwapi.lib dbghelp.lib version.lib
  )
  target_link_options(TerraExecutable
    PRIVATE
      ${TERRA_EXPORTS}
  )
endif()

install(
  TARGETS TerraLibrary
  EXPORT TerraExports
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

install(
  FILES ${PROJECT_BINARY_DIR}/include/terra/terra.h
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/terra
)

install(
  FILES ${PROJECT_SOURCE_DIR}/release/share/terra/LICENSE.txt
  DESTINATION ${CMAKE_INSTALL_DATADIR}/terra
)
install(
  FILES ${PROJECT_SOURCE_DIR}/release/share/terra/README.md
  DESTINATION ${CMAKE_INSTALL_DATADIR}/terra
)
