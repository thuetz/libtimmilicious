CMAKE_MINIMUM_REQUIRED( VERSION 2.6 )
PROJECT( libTimmilicious )

# set the current verison of the library
SET( TIMMI_LIB_SOVERSION "2" ) # should just be changed if the ABI changes

# setup the path where CMake should search for Find*.cmake scripts
SET( CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}/cmake" )

# find and configure the required libraries
FIND_PACKAGE( Boost COMPONENTS system thread timer filesystem REQUIRED )
FIND_PACKAGE( HDF5 REQUIRED )
FIND_PACKAGE( OpenCV REQUIRED )

#
option( BUILD_TESTS "" OFF )
if( BUILD_TESTS )
	find_package( GTest REQUIRED )
	add_executable( tests src/tests/progressBar.cxx src/tests/hdf5.cxx )
	target_link_libraries( tests timmilicious ${GTEST_LIBRARIES} ${GTEST_MAIN_LIBRARY} )
	set_target_properties( tests PROPERTIES COMPILE_FLAGS "-std=c++11 -Wno-global-constructors" )
endif( BUILD_TESTS )

#
option( BUILD_EXAMPLES "" OFF )

# generate a list of all source files of the library
SET( LIBTIMMI_SOURCE_FILES ${LIBTIMMI_SOURCE_FILES} "${PROJECT_BINARY_DIR}/timmilicious.cxx" )
SET( LIBTIMMI_SOURCE_FILES ${LIBTIMMI_SOURCE_FILES} src/timmilicious/ui/ProgressBar.cxx )
SET( LIBTIMMI_SOURCE_FILES ${LIBTIMMI_SOURCE_FILES} src/timmilicious/io/HDF5.cxx )

# generate a list of all header files of the library
SET( LIBTIMMI_HEADER_FILES ${LIBTIMMI_HEADER_FILES} src/timmilicious/timmilicious.hxx )
SET( LIBTIMMI_HEADER_FILES ${LIBTIMMI_HEADER_FILES} src/timmilicious/ui/ProgressBar.hxx )
SET( LIBTIMMI_HEADER_FILES ${LIBTIMMI_HEADER_FILES} src/timmilicious/io/HDF5.hxx )

# set flags to get clean code (at least on UNIX platforms)
if( "${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU" )
	set( CMAKE_CXX_FLAGS "-std=c++11 -Wall -Werror -pedantic" CACHE STRING "Flags used by the compiler during all build types." FORCE )
elseif ( "${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang" )
	set( CMAKE_CXX_FLAGS "-std=c++11 -Wall -Werror -pedantic -Weverything -pedantic-errors -Wno-c++98-compat -Wno-c++98-compat-pedantic" CACHE STRING "Flags used by the compiler during all build types." FORCE )
else()
	message( FATAL_ERROR "You are using a non-supported compiler!" )
endif()

#
SET(CMAKE_MODULES_INSTALL_DIR ${CMAKE_ROOT}/Modules CACHE FILEPATH "Find script and CMake Config file will be installed to this directory")

# add the include directories for building this library
INCLUDE_DIRECTORIES( "${CMAKE_CURRENT_SOURCE_DIR}/src" )
INCLUDE_DIRECTORIES( ${Boost_INCLUDE_DIR} )

# make a version file containing the current version from git.
include(GetGitRevisionDescription)
git_describe(VERSION "--tags")

# parse the version information into pieces.
string(REGEX REPLACE "^v([0-9]+)\\..*" "\\1" VERSION_MAJOR "${VERSION}")
string(REGEX REPLACE "^v[0-9]+\\.([0-9]+).*" "\\1" VERSION_MINOR "${VERSION}")
string(REGEX REPLACE "^v[0-9]+\\.[0-9]+\\.([0-9]+).*" "\\1" VERSION_PATCH "${VERSION}")
string(REGEX REPLACE "^v[0-9]+\\.[0-9]+\\.[0-9]+(.*)" "\\1" VERSION_SHA1 "${VERSION}")
set(VERSION_SHORT "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}")

# define a macro to install the header file properly
MACRO(INSTALL_HEADERS_WITH_DIRECTORY HEADER_LIST)
FOREACH(HEADER ${${HEADER_LIST}})
    STRING( LENGTH ${HEADER} HEADER_LEN )
    MATH( EXPR HEADER_LEN "${HEADER_LEN}-4" )
    STRING( SUBSTRING ${HEADER} 4 ${HEADER_LEN} HEADER2 )
    STRING(REGEX MATCH "(.*)[/\\]" DIR ${HEADER2})
    INSTALL(FILES ${HEADER} DESTINATION include/${DIR})
ENDFOREACH(HEADER)
ENDMACRO(INSTALL_HEADERS_WITH_DIRECTORY)

#
CONFIGURE_FILE( "${PROJECT_SOURCE_DIR}/src/timmilicious/timmilicious.cxx.in" "${PROJECT_BINARY_DIR}/timmilicious.cxx" )
CONFIGURE_FILE( "${PROJECT_SOURCE_DIR}/dist/arch/PKGBUILD.in" "${PROJECT_SOURCE_DIR}/dist/arch/PKGBUILD" )

# specify how to build the library
ADD_LIBRARY( timmilicious SHARED ${LIBTIMMI_SOURCE_FILES} ${LIBTIMMI_HEADER_FILES} ) 
TARGET_LINK_LIBRARIES( timmilicious ${Boost_LIBRARIES} ${HDF5_LIBRARIES} ${OpenCV_LIBRARIES} )
SET_TARGET_PROPERTIES( timmilicious PROPERTIES VERSION ${VERSION_SHORT} SOVERSION ${TIMMI_LIB_SOVERSION} )

# build the testing application
if( BUILD_EXAMPLES )
	add_executable( timmitest_progressbar src/examples/progressBar.cxx )
	target_link_libraries( timmitest_progressbar timmilicious )
endif( BUILD_EXAMPLES )

# define the install actions to perform
INSTALL( TARGETS timmilicious DESTINATION lib )
INSTALL( FILES FindTimmilicious.cmake DESTINATION ${CMAKE_MODULES_INSTALL_DIR} )
INSTALL_HEADERS_WITH_DIRECTORY( LIBTIMMI_HEADER_FILES )
