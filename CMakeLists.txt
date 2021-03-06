# Top level CMakeLists.txt
#
# minimum required cmake version
cmake_minimum_required( VERSION 3.12.0 FATAL_ERROR )

# set cmake policy
if( NOT CMAKE_VERSION VERSION_LESS 3.12.0 )
  # Use latest policy
  cmake_policy( VERSION ${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION} )
endif()

# project name
project( vvdec )

# enable or disable Boost
set( VVDEC_ENABLE_BITSTREAM_DOWNLOAD OFF CACHE BOOL "Enable or disable bitstream download" )

# enable install target
set( VVDEC_ENABLE_INSTALL ON CACHE BOOL "Enable or disable install target" )


# enable or disable Intel Vtune ITT Tracing
#if( CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo" )
#    set( VVDEC_ENABLE_ITT ON CACHE BOOL "Enable Intel Runtime Support for Profiling" )
#else()
#    set( VVDEC_ENABLE_ITT OFF CACHE BOOL "Enable Intel Runtime Support for Profiling" )
#endif()

# set default CMAKE_BUILD_TYPE to Release if not set
if( NOT CMAKE_BUILD_TYPE )
  set( CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel." FORCE )
endif()

#if( CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR
#    CMAKE_CXX_COMPILER_ID STREQUAL "Clang" )
#  set( USE_ADDRESS_SANITIZER OFF CACHE BOOL "Compiles and links with -fsanitize=address" )
#endif()

#set( ENABLE_LINK_TIME_OPT     ON       CACHE BOOL   "Enable link time optimization for release and profile builds" )
#set( OPT_TARGET_ARCH         "native"  CACHE STRING "Create code and optimize for this architecture (default native)" )
#set( DISABLE_BOOST_DEPENDENCY OFF      CACHE BOOL   "Disable Boost dependency => disables the test suite" )

#set( SET_ENABLE_TRACING OFF CACHE BOOL "Set ENABLE_TRACING as a compiler flag" )
#set( ENABLE_TRACING OFF CACHE BOOL "If SET_ENABLE_TRACING is on, it will be set to this value" )

# set c++14
set( CMAKE_CXX_STANDARD 14 )
set( CMAKE_CXX_STANDARD_REQUIRED ON )

# compile everything position independent (even static libraries)
set( CMAKE_POSITION_INDEPENDENT_CODE TRUE )

# set verbose compile options
#set( CMAKE_VERBOSE_MAKEFILE ON )

# use folders in IDEs for projects (e.g. lib sample app test)
set_property( GLOBAL PROPERTY USE_FOLDERS ON )

set( CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/modules" )
message( STATUS "CMAKE_MODULE_PATH: updating module path to: ${CMAKE_MODULE_PATH}" )

# Enable multithreading
find_package( Threads REQUIRED )

# enable sse4.1 build for all source files for gcc and clang
if( UNIX OR MINGW )
  add_compile_options( "-msse4.1" )
endif()

# set _WIN32_WINNT
if( WIN32 )
  # set _WIN32_WINT version global
  add_definitions( -D_WIN32_WINNT=0x0600 )
endif()

# enable parallel build for Visual Studio
if( MSVC )
  # add compile options
  add_compile_options( "/MP" )
  add_compile_options( "/EHsc" )
endif()

## set address sanitizer compiler arguments
#if( USE_ADDRESS_SANITIZER )
#  # add compile options
#  add_compile_options( "-fsanitize=address" )
#  add_link_options( "-fsanitize=address" )
#endif()

# use ccache
find_program( CCACHE_FOUND ccache )
if( CCACHE_FOUND )
  message( STATUS "ccache found. using it." )
  set_property( GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache )
  set_property( GLOBAL PROPERTY RULE_LAUNCH_LINK ccache )
endif()

if( VVDEC_ENABLE_ITT )
  if( MSVC )
    set( ITT_PATH "c:/Program Files (x86)/IntelSWTools/VTune Amplifier/" CACHE STRING "Path to the installation directory of Intel VTunes" )
  elseif( APPLE )
    message( WARNING "Not yet supported on Mac OS X" )
  elseif( UNIX OR MINGW )
    if( EXISTS "/opt/intel/vtune_profiler" )
      set( ITT_PATH "/opt/intel/vtune_profiler" CACHE STRING "Path to the installation directory of Intel VTunes" )
    else()
      set( ITT_PATH "/opt/intel/vtune_amplifier" CACHE STRING "Path to the installation directory of Intel VTunes" )
    endif()
  endif()

  if( EXISTS ${ITT_PATH} )
    set( LIB_NAME INTEL_ITT )
    add_library( ${LIB_NAME} STATIC IMPORTED GLOBAL )
    if( MSVC )
      set_target_properties( ${LIB_NAME} PROPERTIES IMPORTED_LOCATION        ${ITT_PATH}/lib64/libittnotify.lib )
    elseif( APPLE )
      # not supported
    elseif( UNIX OR MINGW )
      set_target_properties( ${LIB_NAME} PROPERTIES IMPORTED_LOCATION        ${ITT_PATH}/lib64/libittnotify.a )
      set_target_properties( ${LIB_NAME} PROPERTIES INTERFACE_LINK_LIBRARIES dl )
    endif()

    # set include directory. relative paths do not work.
    set_target_properties( ${LIB_NAME} PROPERTIES INTERFACE_INCLUDE_DIRECTORIES ${ITT_PATH}/include )
    set( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DTRACE_ENABLE_ITT" )
    set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DTRACE_ENABLE_ITT" )
  else()
    message( WARNING "ITT_PATH ${ITT_PATH} not found, ignoring option VVCLib_ENABLE_ITT" )
  endif()
endif()

#if( ENABLE_LINK_TIME_OPT )
#    set( CMAKE_INTERPROCEDURAL_OPTIMIZATION_RELEASE ON )
#    set( CMAKE_INTERPROCEDURAL_OPTIMIZATION_RELWITHDEBINFO ON )
#    set( CMAKE_INTERPROCEDURAL_OPTIMIZATION_MINSIZEREL ON )
#endif()

#if( (UNIX OR MINGW) AND NOT OPT_TARGET_ARCH STREQUAL "" )
#    set( CMAKE_C_FLAGS   "${CMAKE_C_FLAGS} -march=${OPT_TARGET_ARCH} -mtune=${OPT_TARGET_ARCH}" )
#    set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=${OPT_TARGET_ARCH} -mtune=${OPT_TARGET_ARCH}" )
#endif()

#if( SET_ENABLE_TRACING )
#  if( ENABLE_TRACING )
#    add_compile_definitions( ENABLE_TRACING=1 )
#  else()
#    add_compile_definitions( ENABLE_TRACING=0 )
#  endif()
#endif()

add_subdirectory( "source/Lib/vvdec" )
add_subdirectory( "source/App/vvdecapp" )

# set directory global
set( BITSTREAM_INSTALL_DIR_BASE "${CMAKE_SOURCE_DIR}/ext/bitstreams" )

if( VVDEC_ENABLE_BITSTREAM_DOWNLOAD )
  # enable FetchContent
  include( FetchContent )
  
  # get bitstreams
  set( BITSTREAM_URL_BASE "https://www.itu.int/wftp3/av-arch/jvet-site/bitstream_exchange/VVC/under_test/VTM-10.0" )
  
  # create a list of all test bitstreams
  list( APPEND BITSTREAM_FILES
    "JCCR_A_Nokia_2.zip" "JCCR_B_Nokia_2.zip"
    "APSLMCS_A_Dolby_3.zip" "APSLMCS_B_Dolby_3.zip" "APSLMCS_C_Dolby_2.zip"
    "BDPCM_A_Orange_2.zip"
    "DEBLOCKING_Ericsson_2.zip"
    "DPB_A_Sharplabs_2.zip" "DPB_B_Sharplabs_2.zip"
    # "MCS_A_Dolby_3.zip" "MCS_B_Dolby_2.zip" # fails
    "MPM_A_LGE_3.zip"
    "POUT_A_Sharplabs_2.zip"
    "PPS_A_Bytedance_1.zip" "PPS_B_Bytedance_1.zip" "PPS_C_Bytedance_1.zip"
    "SDH_A_Dolby_2.zip"
    "SPS_A_Bytedance_1.zip" "SPS_B_Bytedance_1.zip" "SPS_C_Bytedance_1.zip"
    "VUI_A_Dolby_2.zip" )
  
  # download and uncompress
  foreach( BITSTREAM_FILE IN LISTS BITSTREAM_FILES )
    string( REGEX REPLACE ".*/(.*)\\..*" "\\1" SUB_DIR "${BITSTREAM_URL_BASE}/${BITSTREAM_FILE}" )
    set( BITSTREAM_INSTALL_DIR "${BITSTREAM_INSTALL_DIR_BASE}/${SUB_DIR}" )
    if( NOT EXISTS "${BITSTREAM_INSTALL_DIR}" )
      FetchContent_Populate( ${SUB_DIR}_download SOURCE_DIR ${BITSTREAM_INSTALL_DIR} URL "${BITSTREAM_URL_BASE}/${BITSTREAM_FILE}" )  
    endif()  
  endforeach()
endif()

# enable testing with ctest
enable_testing()

# find all bitstreams and add test
file( GLOB_RECURSE BITSTREAMS LIST_DIRECTORIES false ${BITSTREAM_INSTALL_DIR_BASE}/*.bit )
foreach( BITSTREAM IN LISTS BITSTREAMS )
  string( REGEX REPLACE ".*/(.*)" "\\1" BITSTREAM_SHORT "${BITSTREAM}" )
  add_test( NAME Test_${BITSTREAM_SHORT} COMMAND vvdecapp -b ${BITSTREAM} -dph )
endforeach()

if( VVDEC_ENABLE_INSTALL )
  set( CMAKE_INSTALL_PREFIX "${CMAKE_SOURCE_DIR}/install" CACHE PATH "Standard install prefix" FORCE )
  include( cmake/modules/VVDecInstall.cmake )
endif()
