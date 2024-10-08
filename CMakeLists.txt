
# CMake code for building the plugin and possible also testcode in the testcode/
# subdirectory. In general, please try to refrain from editing this file!
# Anything special that needs to be done to support the plugin development
# environment can be done in testcode/CMakeLists.txt, so it won't affect the main plugin.

cmake_minimum_required(VERSION 3.10...3.19)

execute_process( COMMAND ncrystal-config --show cmakedir
                 OUTPUT_VARIABLE NCrystal_DIR
                 OUTPUT_STRIP_TRAILING_WHITESPACE )
find_package(NCrystal REQUIRED)

#Extract plugin name from ncplugin_name.txt:
file(STRINGS "${CMAKE_CURRENT_LIST_DIR}/ncplugin_name.txt" NCPlugin_NAME LIMIT_COUNT 1)
string(STRIP "${NCPlugin_NAME}" NCPlugin_NAME)

project( "NCPlugin_${NCPlugin_NAME}" VERSION 0.0.1 LANGUAGES CXX)

#Project has two options. While developing the project, one should use the default values.

#1) NCPLUGIN_DEVMODE: enables the subproject in testcode/ and enables various
#   strict compilation options, which helps to ensure that the plugin will
#   contain code appropriate for NCrystal.
#2) NCPLUGIN_ASBUILTIN: This option is used when the plugin code should be
#   embedded directly into the primary NCrystal library.

option(NCPLUGIN_DEVMODE "Enable strict compilation flags and build test code" ON)
option(NCPLUGIN_ASBUILTIN "Do not build plugin, just prepare CMake variables for static inclusion (variable used by NCrystal's CMake code)" OFF)

if (NCPLUGIN_DEVMODE AND NCPLUGIN_ASBUILTIN)
  message(FATAL_ERROR "The options NCPLUGIN_ASBUILTIN and NCPLUGIN_DEVMODE can not be enabled simultaneously")
endif()

#Ensure we have a default build type if not already specified:
if ( NOT DEFINED CMAKE_BUILD_TYPE )
  if (NCPLUGIN_DEVMODE)
    set( CMAKE_BUILD_TYPE Debug )
  else()
    set( CMAKE_BUILD_TYPE RelWithDebInfo )
  endif()
endif()

#Plugin C++ files:
function(srcfileglob varname pattern)
  #Glob with CONFIGURE_DEPENDS + ignoring temporary files left around by editors.
  file( GLOB tmpall LIST_DIRECTORIES false CONFIGURE_DEPENDS "${PROJECT_SOURCE_DIR}/${pattern}" )
  set(tmp "")
  foreach(fn ${tmpall})
    get_filename_component( bn "${fn}" NAME)
    if (bn MATCHES "(#|~| )+")#could ignore . as well for scripts "|\\."
      message("----> Ignoring file with invalid name: ${bn}")
    else()
      list(APPEND tmp "${fn}")
    endif()
  endforeach()
  set( ${varname} ${tmp} PARENT_SCOPE )
endfunction()

#Find files (also has the effect of triggering auto-reconf if the glob results change):
srcfileglob( plugin_srcfiles "src/*.cc" )
srcfileglob( plugin_hdrfiles  "include/*.hh" )
srcfileglob( plugin_hdrfiles_icc  "include/*.icc" )
srcfileglob( plugin_datafiles  "data/*.ncmat" )
list(APPEND plugin_hdrfiles ${plugin_hdrfiles_icc})
srcfileglob( dummy "src/*.hh" )#To trigger reconf
srcfileglob( dummy "src/*.icc" )#To trigger reconf

if (NCPLUGIN_ASBUILTIN)
  #Special mode which does almost nothing.
  #NCrystal's main cmake code will build the plugin code into the primary
  #NCrystal library. For that to work, we need to ensure that the files from the
  #current plugin get the correct compile definitions:
  get_directory_property(hasParent PARENT_DIRECTORY)
  if (NOT hasParent)
    message(FATAL_ERROR "NCPLUGIN_ASBUILTIN option can only be used when the plugin project is a sub-project")
  endif()
  set(NCPLUGIN_SRCFILES ${plugin_srcfiles} PARENT_SCOPE)#<--- pass up variable to parent project
  set(NCPLUGIN_DATAFILES ${plugin_datafiles} PARENT_SCOPE)#<--- pass up variable to parent project
  set(NCPLUGIN_COMPILEDEFS NCPLUGIN_NAME=${NCPlugin_NAME} NCPLUGIN_ASBUILTIN PARENT_SCOPE)
  set(NCPLUGIN_INCDIRS ${PROJECT_SOURCE_DIR}/include PARENT_SCOPE)
  return()
endif()

#Standard build, need NCrystal as dependency:

find_package(NCrystal 3.0.0 REQUIRED)

if ( NCPLUGIN_DEVMODE )
  #Compile with very strict C++11 options during normal development (this gives
  #better quality code, and ensures that the code developed is better suited for
  #possible inclusion in NCrystal at some point):
  set(CMAKE_CXX_STANDARD 11)
  set(CMAKE_CXX_STANDARD_REQUIRED ON)
  set(CMAKE_CXX_EXTENSIONS OFF)
  set(CMAKE_CXX_COMPILE_FLAGS ${CMAKE_CXX_COMPILE_FLAGS} -Wall -Wextra -pedantic -Werror )
  #Limit how many errors are dumped on the poor developer:
  include(CheckCXXCompilerFlag)
  check_cxx_compiler_flag( -fmax-errors=3 COMPILER_SUPPORTS_MAXERRORS )
  if ( COMPILER_SUPPORTS_MAXERRORS )
    set(CMAKE_CXX_COMPILE_FLAGS ${CMAKE_CXX_COMPILE_FLAGS} -fmax-errors=3 )
  endif()
  #Try to make life easier by adding rpaths (to ncrystal lib):
  if ( NOT DEFINED CMAKE_INSTALL_RPATH_USE_LINK_PATH )
    set( CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE )
  endif()
endif()

#The actual plugin should be build as a shared library. In case it is ever
#needed, we export everything for possible downstream usage: header files,
#targets, and cmake cfg files:
add_library( pluginlib SHARED ${plugin_srcfiles} )
target_compile_definitions( pluginlib PRIVATE NCPLUGIN_NAME=${NCPlugin_NAME} )
set_target_properties( pluginlib PROPERTIES OUTPUT_NAME ${PROJECT_NAME} )
target_compile_definitions( pluginlib PRIVATE NCRYSTAL_NO_CMATH_CONSTANTS )
target_link_libraries( pluginlib PUBLIC NCrystal::NCrystal )
target_include_directories( pluginlib PRIVATE "${PROJECT_SOURCE_DIR}/src"
  PUBLIC $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include/${PROJECT_NAME}> )
install( TARGETS pluginlib EXPORT ${PROJECT_NAME}Targets DESTINATION lib )
install( FILES ${plugin_hdrfiles} DESTINATION include/${PROJECT_NAME} )
install( EXPORT ${PROJECT_NAME}Targets FILE "${PROJECT_NAME}Targets.cmake" NAMESPACE "${PROJECT_NAME}::" DESTINATION lib/cmake )
add_library("${PROJECT_NAME}::pluginlib" ALIAS pluginlib)
include(CMakePackageConfigHelpers)
write_basic_package_version_file( "${PROJECT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
  VERSION ${PROJECT_VERSION} COMPATIBILITY AnyNewerVersion )
configure_file( "${PROJECT_SOURCE_DIR}/PkgConfig.cmake.in"
  "${PROJECT_BINARY_DIR}/${PROJECT_NAME}Config.cmake" @ONLY )
install( FILES "${PROJECT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
  "${PROJECT_BINARY_DIR}/${PROJECT_NAME}Config.cmake" DESTINATION lib/cmake )

#For simplicity, any data files are embedded directly in the library (except in
#NCPLUGIN_DEVMODE where the testcode project will make them available via
#symlinks):

if (plugin_datafiles)
  #In any mode we sanity check the names of the data files (so developers will
  #notice and fix any issues):
  foreach(df ${plugin_datafiles})
    get_filename_component(dfbn "${df}" NAME)
    if(NOT dfbn MATCHES "ncplugin-${NCPlugin_NAME}_.+\.ncmat")
      message(FATAL_ERROR "ERROR: name of exported datafile ${dfbn} does not have required form: ncplugin-${NCPlugin_NAME}_*.ncmat")
    endif()
  endforeach()
  if (NOT NCPLUGIN_DEVMODE)
    #Check that data file name follows convention: ncplugin-<pluginname>_*.ncmat:
    if (NOT NCrystal_CMD_NCMAT2CPP)
      message(FATAL_ERROR "ERROR: NCrystal installation does not provide ncrystal_ncmat2cpp command which is needed to embed datafiles in plugin.")
    endif()
    #Generate C++ code from the .ncmat files:
    execute_process(COMMAND "${NCrystal_CMD_NCMAT2CPP}"
      "-n" "NCPluginNamespace::registerDataFiles"
      --include NCrystal/NCPluginBoilerplate.hh
      "-o" "${PROJECT_BINARY_DIR}/autogen_${NCPlugin_NAME}_ncmat_data.cc" ${plugin_datafiles} RESULT_VARIABLE status )
    if(status AND NOT status EQUAL 0)
      message(FATAL_ERROR "Failure while trying to invoke ncrystal_ncmat2cpp (from ${NCrystal_CMD_NCMAT2CPP}).")
    endif()
    target_sources(pluginlib PRIVATE "${PROJECT_BINARY_DIR}/autogen_${NCPlugin_NAME}_ncmat_data.cc")#too late to just append to plugin_srcfiles
    target_compile_definitions(pluginlib PRIVATE NCPLUGIN_DO_REGISTERDATAFILES)
    message("-- Generated autogen_${NCPlugin_NAME}_ncmat_data.cc with embedded NCMAT data (will be compiled into the plugin library).")
  endif()
endif()

if ( NCPLUGIN_DEVMODE )
  #Development code:
  add_subdirectory(testcode)
endif()
