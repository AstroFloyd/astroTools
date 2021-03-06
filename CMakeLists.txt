##
##  CMakeFile for astroTools
##  AF, 2015-04-20
##
##  Copyright 2007-2015 AstroFloyd - astrofloyd.org
##  
##  This file is part of the astroTools package,
##  see: http://astrotools.sf.net/
##  
##  astroTools is free software: you can redistribute it and/or modify
##  it under the terms of the GNU General Public License as published by
##  the Free Software Foundation, either version 3 of the License, or
##  (at your option) any later version.
##  
##  astroTools is distributed in the hope that it will be useful,
##  but WITHOUT ANY WARRANTY; without even the implied warranty of
##  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
##  GNU General Public License for more details.
##  
##  You should have received a copy of the GNU General Public License
##  along with astroTools.  If not, see <http://www.gnu.org/licenses/>.
##
##  To install this software on your system, see the INSTALL file
##  in the root or doc/ directory of this package.
##


cmake_minimum_required( VERSION 2.8 )

# Set build type. Do this *before* we set the project name:
if( NOT CMAKE_BUILD_TYPE )
  set( CMAKE_BUILD_TYPE Release CACHE STRING
    "Choose the type of build, options are: None Debug Release RelWithDebInfo Profile."
    FORCE )
endif( NOT CMAKE_BUILD_TYPE )
set( CMAKE_CONFIGURATION_TYPES "${CMAKE_BUILD_TYPE}" CACHE INTERNAL "internal" )


# Project name and language:
project( astroTools Fortran )


# Search in the CMake/ directory for CMake modules:
list( APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/CMake )


# Various compile/optimisation options that we may want to enable:
include( SetCompileOptions )


# Get the compiler name (for compiler flags and to find libraries):
get_filename_component( Fortran_COMPILER_NAME ${CMAKE_Fortran_COMPILER} NAME )



# Find dependencies:
find_package( LibSUFR REQUIRED )
find_package( LibTheSky REQUIRED )
set( INCLUDE_FLAGS "-I${LibSUFR_INCLUDES} -I${LibTheSky_INCLUDES}" )  # will be transferred to CompilerFlags



# Set FORTRAN compiler flags:
include( CompilerFlags_Fortran )


# Create the file code_version.f90, which contains the code version number/hash and date:
if( NOT EXISTS ${CMAKE_SOURCE_DIR}/src/code_version.f90 OR CREATE_VERSION )
  # Code version generator:
  add_custom_command(
    OUTPUT ${CMAKE_SOURCE_DIR}/src/code_version.f90
    COMMAND cd $(CMAKE_SOURCE_DIR)
    COMMAND . ${CMAKE_SOURCE_DIR}/code_version.sh $(CMAKE_SOURCE_DIR) src/code_version.f90 ${Fortran_COMPILER_NAME} ${OPT_FLAGS}
    )
  
  # Tell CMake the source won't be available until build time:
  set_source_files_properties( ${CMAKE_SOURCE_DIR}/src/code_version.f90 PROPERTIES GENERATED 1 )
endif( NOT EXISTS ${CMAKE_SOURCE_DIR}/src/code_version.f90 OR CREATE_VERSION )


# Find the package version from the VERSION file and stick it into PKG_VERSION:
# The file VERSION is either in the root directory of the package, or in doc/:
set( VERSION_FILE ${CMAKE_SOURCE_DIR}/VERSION )
if( NOT EXISTS ${VERSION_FILE} )
  set( VERSION_FILE ${CMAKE_SOURCE_DIR}/doc/VERSION )
endif( NOT EXISTS ${VERSION_FILE} )

set( PKG_VERSION "Unknown" )   # Indicates that the version number was not found
file(STRINGS ${VERSION_FILE} Lines )
foreach( Line ${Lines} )
  string(REGEX MATCH "Release version:.+" Line ${Line} )  # Returns the matching string
  string(COMPARE NOTEQUAL "${Line}" "" Matches )          # If the string is not empty, we have a match
  if( Matches )
    string(REPLACE "Release version:" "" Line ${Line})    # Remove text
    string(STRIP ${Line} Line)                            # Strip leading and trailing spaces
    set( PKG_VERSION ${Line} )
  endif( Matches )
endforeach()
message( STATUS "astroTools version reported by the package: ${PKG_VERSION}" )
message( STATUS "Compiling for an ${CMAKE_SYSTEM_PROCESSOR} architecture" )
message( STATUS "" )


# Put common source files in a static library, and link it to the other object files:
add_library( "At"  STATIC  src/functions.f90 src/code_version.f90 )       # Creates libAt.a
target_link_libraries( At  ${LibSUFR_LIBRARIES} ${LibTheSky_LIBRARIES} )  # link libSUFR and libTheSky to libAt.a


# List of programs to compile:
set( AT_PROGRAMS 
  add_magnitudes arcsec2deg_rad azimuth2winddir
  cal2gps cal2jd cal2times cal2unix comet_eq2ap
  date_time_calc dd2dms_rad distance_in_sky_ecl distance_in_sky_eq dms2dd_rad
  eq2hor gps2cal
  jd2cal jd2times
  planetdata rad2dd_dms rect2spher settings time2hms unix2cal
  )


# For each program, specify the binary name (at_PROGRAM), source name (src/PROGRAM.f90), and compile and link the code:
foreach( program ${AT_PROGRAMS} )
  add_executable( at_${program} "src/${program}.f90")
  target_link_libraries( at_${program}  ${LibSUFR_LIBRARIES} ${LibTheSky_LIBRARIES} At )
  set( AT_BINARIES ${AT_BINARIES} at_${program} )          # For install()
endforeach( program )


# Install the binaries:
install( TARGETS  ${AT_BINARIES}  RUNTIME DESTINATION bin )

# Install documents:
if( NOT EXISTS VERSION )
  install( FILES doc/CHANGELOG doc/LICENCE doc/README doc/VERSION DESTINATION ${CMAKE_INSTALL_PREFIX}/share/doc/astrotools-${PKG_VERSION} )
else( NOT EXISTS VERSION )
  install( FILES CHANGELOG LICENCE README VERSION DESTINATION ${CMAKE_INSTALL_PREFIX}/share/doc/astrotools-${PKG_VERSION} )
endif( NOT EXISTS VERSION )

# Install man pages:
install( DIRECTORY man/ DESTINATION ${CMAKE_INSTALL_PREFIX}/share/man )

