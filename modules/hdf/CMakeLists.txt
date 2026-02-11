set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR})

if(WIN32)
  # windows cmake internal lookups are broken for now
  # will lookup for headers and shared libs given HDF_DIR env
  find_path(HDF5_INCLUDE_DIRS hdf5.h HINTS "$ENV{HDF5_DIR}\\..\\include")
  find_library(HDF5_C_LIBRARY NAMES hdf5 HINTS "$ENV{HDF5_DIR}\\..\\lib")
  if(HDF5_INCLUDE_DIRS AND HDF5_C_LIBRARY)
    set(HDF5_FOUND "YES")
    set(HDF5_LIBRARIES ${HDF5_C_LIBRARY})
    mark_as_advanced(HDF5_LIBRARIES)
    mark_as_advanced(HDF5_C_LIBRARY)
    mark_as_advanced(HDF5_INCLUDE_DIRS)
    add_definitions(-DH5_BUILT_AS_DYNAMIC_LIB -D_HDF5USEDLL_)
  else()
     set(HDF5_FOUND "NO")
  endif()
else()
  if(NOT CMAKE_CROSSCOMPILING) # iOS build should not reuse OSX package
    find_package(HDF5)
  endif()
endif()

if(NOT HDF5_FOUND)
  ocv_module_disable(hdf)
  return()
endif()

# Check for HDF5 float16 (IEEE F16) support (available since HDF5 1.14.3)
include(CheckCSourceCompiles)
set(_cmake_required_includes_save ${CMAKE_REQUIRED_INCLUDES})
set(_cmake_required_libraries_save ${CMAKE_REQUIRED_LIBRARIES})
set(CMAKE_REQUIRED_INCLUDES ${HDF5_INCLUDE_DIRS})
set(CMAKE_REQUIRED_LIBRARIES ${HDF5_LIBRARIES})
check_c_source_compiles("
#include <hdf5.h>
int main() {
    hid_t t = H5T_IEEE_F16LE;
    (void)t;
    return 0;
}
" HAVE_HDF5_IEEE_F16)
set(CMAKE_REQUIRED_INCLUDES ${_cmake_required_includes_save})
set(CMAKE_REQUIRED_LIBRARIES ${_cmake_required_libraries_save})

if(HAVE_HDF5_IEEE_F16)
  message(STATUS "HDF5: float16 (IEEE F16) support detected")
  add_definitions(-DHAVE_HDF5_F16)
else()
  message(STATUS "HDF5: float16 (IEEE F16) support not available (requires HDF5 >= 1.14.3)")
endif()

ocv_define_module(hdf opencv_core WRAP python)
ocv_warnings_disable(CMAKE_CXX_FLAGS -Winvalid-offsetof)

set(the_description "Hierarchical Data Format I/O")

ocv_target_link_libraries(${the_module} ${HDF5_LIBRARIES})
ocv_include_directories(${HDF5_INCLUDE_DIRS})
