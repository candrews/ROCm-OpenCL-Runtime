find_package(Clang REQUIRED CONFIG)

# FIXME: CLANG_CMAKE_DIR seems like the most stable way to find this, but
# really there is no way to reliably discover this header.
#
# We effectively back up to the Clang output directory (for the case of a build
# tree) or install prefix (for the case of an installed copy), and then search
# for a file named opencl-c.h anywhere below that. We take the first result in
# the case where there are multiple (e.g. if there is an installed copy nested
# in a build directory). This is a bit imprecise, but it covers cases like MSVC
# adding some additional configuration-specific subdirectories to the build
# tree but not to an installed copy.
file(GLOB_RECURSE OPENCL_C_H_LIST "${CLANG_CMAKE_DIR}/../../../*/opencl-c.h")
list(GET OPENCL_C_H_LIST 0 OPENCL_C_H)
if (NOT EXISTS "${OPENCL_C_H}" OR IS_DIRECTORY "${OPENCL_C_H}")
  message(FATAL_ERROR "Unable to locate opencl-c.h from the supplied Clang. The path '${CLANG_CMAKE_DIR}/../../../*' was searched.")
endif()

include(bc2h)

set(INC_SUFFIX "amdgcn.inc")

include_directories(${CMAKE_SOURCE_DIR}/compiler/lib/loaders/elf/utils/common)
include_directories(${CMAKE_SOURCE_DIR}/compiler/lib/loaders/elf/utils/libelf)
include_directories(${CMAKE_CURRENT_BINARY_DIR})
include_directories(${ROCM_OCL_INCLUDES})

if(${USE_COMGR_LIBRARY} MATCHES "yes")
  add_definitions(-DUSE_COMGR_LIBRARY -DCOMGR_DYN_DLL)
  find_package(amd_comgr REQUIRED CONFIG)
  include_directories("$<TARGET_PROPERTY:amd_comgr,INTERFACE_INCLUDE_DIRECTORIES>")
endif()

add_library(oclrocm OBJECT
  roccounters.cpp
  rocprintf.cpp
  rocprogram.cpp
  rocmemory.cpp
  rocdevice.cpp
  rocblit.cpp
  rockernel.cpp
  rocvirtual.cpp
  rocglinterop.cpp
  rocappprofile.cpp
  rocsettings.cpp
  rocschedcl.cpp
)
set_target_properties(oclrocm PROPERTIES POSITION_INDEPENDENT_CODE ON)

# generating libraries.amdgcn.inc
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/libraries.amdgcn.inc "// Automatically generated file; DO NOT EDIT.\n")

foreach(AMDGCN_LIB_TARGET ${AMDGCN_LIB_TARGETS})
  get_target_property(lib_file ${AMDGCN_LIB_TARGET} OUTPUT_NAME)
  get_target_property(lib_file_name ${AMDGCN_LIB_TARGET} ARCHIVE_OUTPUT_NAME)
  get_target_property(lib_file_path ${AMDGCN_LIB_TARGET} ARCHIVE_OUTPUT_DIRECTORY)
  set(bclib "${lib_file}")
  set(header "${lib_file_name}.${INC_SUFFIX}")
  set(symbol "${lib_file_name}_lib")
  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${header}
    COMMAND bc2h ${bclib} ${CMAKE_CURRENT_BINARY_DIR}/${header} ${symbol}
    DEPENDS bc2h ${AMDGCN_LIB_TARGET}
    COMMENT "Generating ${header}"
  )
  set_property(DIRECTORY APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES ${CMAKE_CURRENT_BINARY_DIR}/${target}.inc)

  add_custom_target(${header}_target ALL DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${header})
  add_dependencies(oclrocm  ${header}_target)

  file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/libraries.amdgcn.inc "#include \"${header}\"\n")
endforeach()

# Generate function to select libraries for a given GFXIP number.
file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/libraries.amdgcn.inc
  "static inline std::tuple<const char*, const void*, size_t> get_oclc_isa_version(uint gfxip) { \
   switch (gfxip) {\n")
foreach(AMDGCN_LIB_TARGET ${AMDGCN_LIB_TARGETS})
  if (${AMDGCN_LIB_TARGET} MATCHES "^oclc_isa_version_[0-9]+_lib$")
    string(REGEX REPLACE "^oclc_isa_version_([0-9]+)_lib$" "\\1" gfxip ${AMDGCN_LIB_TARGET})
    file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/libraries.amdgcn.inc
      "case ${gfxip}: return std::make_tuple( \"oclc_isa_version_${gfxip}_lib.bc\","
                "  oclc_isa_version_${gfxip}_lib, oclc_isa_version_${gfxip}_lib_size); break;\n")
  endif()
endforeach()
file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/libraries.amdgcn.inc
  "default: return std::make_tuple(\"\",(const void*)0,(size_t)0); }\n}\n")

foreach(AMDGCN_LIB_TARGET ${AMDGCN_LIB_TARGETS})
  if (${AMDGCN_LIB_TARGET} MATCHES "oclc_(.*)_on_lib")
    string(REGEX REPLACE "oclc_(.*)_on_lib" "\\1" function ${AMDGCN_LIB_TARGET})
    file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/libraries.amdgcn.inc
      "static inline std::tuple<const char*,const void*,size_t>  get_oclc_${function}(bool on)"
                " { return std::make_tuple(  on ? \"oclc_${function}_on_lib.bc\" : \"oclc_${function}_off_lib.bc\","
                "  (const void*)(on ? oclc_${function}_on_lib : oclc_${function}_off_lib),"
                " on ? oclc_${function}_on_lib_size : oclc_${function}_off_lib_size);  }\n")
  endif()
endforeach()

# generating opencl*.inc files
add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/opencl1.2-c.amdgcn.pch
  COMMAND clang -cc1 -x cl-header -triple amdgcn-amd-amdhsa -Werror -O3 -DNDEBUG -cl-std=CL1.2 -emit-pch -o ${CMAKE_CURRENT_BINARY_DIR}/opencl1.2-c.amdgcn.pch < ${OPENCL_C_H}
  DEPENDS clang ${OPENCL_C_H}
  COMMENT "Generating opencl1.2-c.amdgcn.pch"
)
add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/opencl1.2-c.amdgcn.inc
  COMMAND bc2h ${CMAKE_CURRENT_BINARY_DIR}/opencl1.2-c.amdgcn.pch ${CMAKE_CURRENT_BINARY_DIR}/opencl1.2-c.amdgcn.inc opencl1_2_c
  DEPENDS bc2h ${CMAKE_CURRENT_BINARY_DIR}/opencl1.2-c.amdgcn.pch
  COMMENT "Generating opencl1.2-c.amdgcn.inc"
)
set_property(DIRECTORY APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES ${CMAKE_CURRENT_BINARY_DIR}/opencl1.2-c.amdgcn.inc ${CMAKE_CURRENT_BINARY_DIR}/opencl1.2-c.amdgcn.pch)
add_custom_target(opencl1.2-c.amdgcn.inc_target ALL DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/opencl1.2-c.amdgcn.inc)
add_dependencies(oclrocm opencl1.2-c.amdgcn.inc_target)

add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/opencl2.0-c.amdgcn.pch
  COMMAND clang -cc1 -x cl-header -triple amdgcn-amd-amdhsa -Werror -O3 -DNDEBUG -cl-std=CL2.0 -emit-pch -o ${CMAKE_CURRENT_BINARY_DIR}/opencl2.0-c.amdgcn.pch < ${OPENCL_C_H}
  DEPENDS clang ${OPENCL_C_H}
  COMMENT "Generating opencl2.0-c.amdgcn.pch"
)
add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/opencl2.0-c.amdgcn.inc
  COMMAND bc2h ${CMAKE_CURRENT_BINARY_DIR}/opencl2.0-c.amdgcn.pch ${CMAKE_CURRENT_BINARY_DIR}/opencl2.0-c.amdgcn.inc opencl2_0_c
  DEPENDS bc2h ${CMAKE_CURRENT_BINARY_DIR}/opencl2.0-c.amdgcn.pch
  COMMENT "Generating opencl2.0-c.amdgcn.inc"
)
set_property(DIRECTORY APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES ${CMAKE_CURRENT_BINARY_DIR}/opencl2.0-c.amdgcn.inc ${CMAKE_CURRENT_BINARY_DIR}/opencl2.0-c.amdgcn.pch)
add_custom_target(opencl2.0-c.amdgcn.inc_target ALL DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/opencl2.0-c.amdgcn.inc)
add_dependencies(oclrocm opencl2.0-c.amdgcn.inc_target)
