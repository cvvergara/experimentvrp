cmake_minimum_required(VERSION 3.2 FATAL_ERROR)

if (${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
  message(FATAL_ERROR "In-source builds not allowed.
  Please make a new directory (called a build directory) and run CMake from there.
  You may need to remove CMakeCache.txt." )
endif()


list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")


#---------------------------------------------
#---------------------------------------------
#---------------------------------------------
# Basic definitions of about the project
#---------------------------------------------
#---------------------------------------------
#---------------------------------------------

project(VRPROUTING VERSION 0.0.0
  LANGUAGES C CXX )
set(PROJECT_VERSION_DEV "")
string(TOLOWER "${PROJECT_NAME}" PROJECT_NAME_LOWER)

set(MINORS 0.0)
set(OLD_SIGNATURES
  )


include(pgr/BuildType)
add_definitions(-DBUILD_TYPE="${CMAKE_BUILD_TYPE}")

include(pgr/GitInfo)
add_definitions(-DPROJECT_GIT_HASH="${PROJECT_GIT_HASH}")

include(pgr/Version)

add_definitions(-DPROJECT_VERSION="v${PROJECT_VERSION}${PROJECT_VERSION_DEV}")
add_definitions(-DPROJECT_LIB_NAME="${PROJECT_NAME_LOWER}-${PROJECT_VERSION}")

string(TIMESTAMP COMPILATION_DATE "\"%Y/%m/%d\"")
add_definitions(-DCOMPILATION_DATE=${COMPILATION_DATE})
add_definitions(-DSYSTEM_NAME="${CMAKE_SYSTEM}")



#=============================================
# Set the working directories
#=============================================
include(pgr/Configure)


#---------------------------------------------
# minimum versions
#---------------------------------------------
#TODO define precisely which versions
set(DOXYGEN_MINIMUM_VERSION "1.7")
set(SPHINX_MINIMUM_VERSION "1.8")
set(POSTGRESQL_MINIMUM_VERSION "10.0.0")
set(BOOST_MINIMUM_VERSION "1.65.0")

message(STATUS "DOXYGEN_MINIMUM_VERSION=${DOXYGEN_MINIMUM_VERSION}")
message(STATUS "SPHINX_MINIMUM_VERSION=${SPHINX_MINIMUM_VERSION}")
message(STATUS "POSTGRESQL_MINIMUM_VERSION=${POSTGRESQL_MINIMUM_VERSION}")
message(STATUS "BOOST_MINIMUM_VERSION=${BOOST_MINIMUM_VERSION}")

# cmake 3.2 adds unwanted flags
if(WIN32 AND MSVC)
  set(CMAKE_C_FLAGS "")
  set(CMAKE_CXX_FLAGS "")
endif()

#-----------------------------------------------------------------------------
#-----------------------------------------------------------------------------
# C/C++ Compiler requirements
#-----------------------------------------------------------------------------
#-----------------------------------------------------------------------------

set(CMAKE_CXX_STANDARD 14)

set(COMPILER_VERSION "\"${CMAKE_CXX_COMPILER_ID}-${CMAKE_CXX_COMPILER_VERSION}\"")
add_definitions(-DCOMPILER_VERSION="\"${COMPILER_VERSION}\"")

#---------------------------------------------
#---------------------------------------------
# Boost
#---------------------------------------------
#---------------------------------------------
find_package(Boost ${BOOST_MINIMUM_VERSION} REQUIRED)
if (NOT Boost_VERSION_MACRO)
  set(Boost_VERSION_MACRO ${Boost_VERSION})
endif()

set(BOOST_VERSION "\"${Boost_MAJOR_VERSION}.${Boost_MINOR_VERSION}.${Boost_SUBMINOR_VERSION}\"")

# To be used in defines
add_definitions(-DBoost_VERSION_MACRO=${Boost_VERSION_MACRO})
# Used in pgr_version
add_definitions(-DBOOST_VERSION=${BOOST_VERSION})
add_definitions(-DBOOST_ALLOW_DEPRECATED_HEADERS)
include_directories(SYSTEM ${Boost_INCLUDE_DIRS})


#---------------------------------------------
# Windows compiler flags
#---------------------------------------------


if(WIN32 AND NOT MSVC)
  set(OS_BUILD  $ENV{OS_BUILD})
  if(NOT OS_BUILD)
    set(OS_BUILD "64")
  endif()
  set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH}:/c/ming${OS_BUILD}/projects/pgx${OS_BUILD}/pg92)
  if (NOT BOOST_ROOT)
    set(BOOST_ROOT c:/ming${OS_BUILD}/msys/local)
  endif()

  if (PROJECT_DEBUG)
    message(STATUS "OS_BUILD=${OS_BUILD}")
    message(STATUS "BOOST_ROOT=${BOOST_ROOT}")
    message(STATUS "CMAKE_PREFIX_PATH=${CMAKE_PREFIX_PATH}")
  endif()
endif(WIN32 AND NOT MSVC)

#-----------------------------------------------------------------------------
#-----------------------------------------------------------------------------
#-----------------------------------------------------------------------------
# Finding prerequisites
#-----------------------------------------------------------------------------
#-----------------------------------------------------------------------------
#-----------------------------------------------------------------------------

#---------------------------------------------
#---------------------------------------------
# Perl
#---------------------------------------------
#---------------------------------------------
find_package(Perl REQUIRED)

#---------------------------------------------
#---------------------------------------------
# PostgreSQL
#---------------------------------------------
#---------------------------------------------
find_package(PostgreSQL ${POSTGRESQL_MINIMUM_VERSION} REQUIRED)
find_program(POSTGRESQL_PG_CONFIG pg_config)
add_definitions(-DPostgreSQL_VERSION_STRING="${PostgreSQL_VERSION_STRING}")

# for XbetaY XalphaY XrcY -> X.Y
string(REGEX REPLACE "^([0-9]+)[beta|alpha|rc|devel].*" "\\1.0" PostgreSQL_VERSION_STRING ${PostgreSQL_VERSION_STRING})
#for X.Y.Z -> XY  Y<10
string(REGEX REPLACE "^([0-9]+)\\.([0-9]+).*" "\\1" PostgreSQL_VERSION_MAYOR ${PostgreSQL_VERSION_STRING})
string(REGEX REPLACE "^([0-9]+)\\.([0-9]+).*" "\\2" PostgreSQL_VERSION_MINOR ${PostgreSQL_VERSION_STRING})
math(EXPR PostgreSQL_VERSION_NUM "${PostgreSQL_VERSION_MAYOR} * 10000 + ${PostgreSQL_VERSION_MINOR}")


# To be used in defines
add_definitions(-DPostgreSQL_VERSION_NUM=${PostgreSQL_VERSION_NUM})

#-------
include_directories(SYSTEM ${PostgreSQL_INCLUDE_DIRS})
if(WIN32)
  include_directories(SYSTEM ${PostgreSQL_INCLUDE_DIRS}/port/win32)
  if(MSVC)
    include_directories(SYSTEM ${POSTGRESQL_INCLUDE_DIR}/port/win32_msvc/)
  endif()
else()
  include_directories(SYSTEM ${PostgreSQL_INCLUDE_DIRS})
endif()


#-------
# extension directory
#-------
execute_process(
  COMMAND ${POSTGRESQL_PG_CONFIG} --sharedir
  OUTPUT_STRIP_TRAILING_WHITESPACE
  OUTPUT_VARIABLE PostgreSQL_EXTENSION_DIR)

#-------
if(PostgreSQL_EXTENSION_DIR)
  set(PostgreSQL_EXTENSION_DIR "${PostgreSQL_EXTENSION_DIR}/extension")
else()
  message(FATAL_ERROR "pg_config --sharedir failed to return a value. Please check your PostgreSQL installation!")
endif()

if (PROJECT_DEBUG)
  message(STATUS "PostgreSQL_FOUND ${PostgreSQL_FOUND}")
  message(STATUS "PostgreSQL_INCLUDE_DIRS ${PostgreSQL_INCLUDE_DIRS}")
  message(STATUS "PostgreSQL_LIBRARIES ${PostgreSQL_LIBRARIES}")
  message(STATUS "PostgreSQL_LIBRARY_DIRS ${PostgreSQL_LIBRARY_DIRS}")
  message(STATUS "PostgreSQL_VERSION_STRING ${PostgreSQL_VERSION_STRING}")
  message(STATUS "PostgreSQL_VERSION_NUM ${PostgreSQL_VERSION_NUM}")
  message(STATUS "POSTGRESQL_PG_CONFIG ${POSTGRESQL_PG_CONFIG}")
  message(STATUS "PGSQL_VERSION=${PGSQL_VERSION}")
endif()



#---------------------------------------------
#---------------------------------------------
# PROJECT includes
#---------------------------------------------
#---------------------------------------------

include_directories(${PROJECT_SOURCE_DIR}/include)

#---------------------------------------------
# Special cases for windows
#---------------------------------------------

if(WIN32)
  link_directories(${POSTGRESQL_LIBRARIES})
  link_libraries(postgres)
endif()

#-----------------------------------------------------------------------------

if (PROJECT_DEBUG)
  get_property(dirs DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY INCLUDE_DIRECTORIES)
  message(STATUS "INCLUDE_DIRECTORIES='${dirs}'")
endif()


#+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
#compiler directives
#+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
# https://www.postgresql.org/docs/10/xfunc-c.html

if(WIN32 AND MSVC)
  add_compile_options(
    -D_CRT_SECURE_NO_DEPRECATE
    -D_SCL_SECURE_NO_DEPRECATE
    -D_CRT_SECURE_NO_WARNINGS
    -D_SCL_SECURE_NO_WARNINGS
    -D_CRT_NONSTDC_NO_DEPRECATE
    -D_CRT_NONSTDC_NO_DEPRECATE
    -EHsc)
else(NOT WIN32)
  add_compile_options(-fPIC)

  if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    add_compile_options(
      "$<$<CONFIG:Debug>:-Wall;-Wextra;-Werror;-Wpedantic;-W;-Wunused;-Wuninitialized;-Wextra;-Wdouble-promotion;-Wconversion>"
      "$<$<CXX_COMPILER_ID:GNU>:-frounding-math>"
      )
  endif()
endif()



#-------------------
# add the subdirectories that have the C/C++ code
#-------------------

foreach (subdir ${PROJECT_SRC_DIRECTORIES})
  add_subdirectory("${PROJECT_SOURCE_DIR}/src/${subdir}")
endforeach()



#-----------------------------------------------------------------------------
#-----------------------------------------------------------------------------
#-----------------------------------------------------------------------------
# PROJECT Library
#-----------------------------------------------------------------------------
#-----------------------------------------------------------------------------
#-----------------------------------------------------------------------------


set(LIBRARY_OUTPUT_DIRECTORY lib)

if(APPLE)
  set(LIBRARY_MODE_TARGET "MODULE")
else(APPLE)
  set(LIBRARY_MODE_TARGET "SHARED")
endif(APPLE)


#-------------------
# pgRouting objects to be linked
#-------------------
foreach (subdir ${PROJECT_SRC_DIRECTORIES} )
  set(PROJECT_OBJECTS ${PROJECT_OBJECTS} "$<TARGET_OBJECTS:${subdir}>")
endforeach()


add_library(${PROJECT_LIB_NAME}
  ${LIBRARY_MODE_TARGET}
  ${PROJECT_OBJECTS})


if(APPLE)
  set_target_properties(${PROJECT_LIB_NAME}
    PROPERTIES
    LINK_FLAGS "-bundle_loader ${POSTGRESQL_EXECUTABLE} -bundle")
endif(APPLE)

if(WIN32 AND MSVC)
  set_target_properties(${PROJECT_LIB_NAME} PROPERTIES PREFIX "lib")
endif(WIN32 AND MSVC)


#-----------------------------------------------------------------------------
#-----------------------------------------------------------------------------
# sql subdirectory creates the files:
#  ${PROJECT_NAME_LOWER}--(version).sql
#  ${PROJECT_NAME_LOWER}--(oldVersion)--(version).sql
#-----------------------------------------------------------------------------
#-----------------------------------------------------------------------------

add_subdirectory(sql)

#-----------------------------------------------------------------------------
#-----------------------------------------------------------------------------
# INSTALLATION
#-----------------------------------------------------------------------------
#-----------------------------------------------------------------------------


install(TARGETS ${PROJECT_LIB_NAME} DESTINATION ${PostgreSQL_LIBRARY_DIRS})
install(FILES
  ${PROJECT_FILES_TO_INSTALL}
  DESTINATION "${PostgreSQL_EXTENSION_DIR}"
  )

#-----------------------------------------------------------------------------
#-----------------------------------------------------------------------------
# DOCUMENTATION
# We are not installing documentation
#-----------------------------------------------------------------------------
#-----------------------------------------------------------------------------
add_subdirectory(doxygen)
add_subdirectory(doc)
