## Based on CMakeLists.txt from AnaEx02 in examples/extended of Geant4.9.6 ##
## Modified for ROOT6 support on 2020/11/26 by Guillaume Pronost
#----------------------------------------------------------------------------
# Setup the project
cmake_minimum_required(VERSION 2.8.10 FATAL_ERROR)
project(WCSim)

if (CMAKE_COMPILER_IS_GNUCC AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.4)
    message(STATUS "GCC version >= 4.4 required!")
    return()
endif()

#----------------------------------------------------------------------------
# Find ROOT (required package) in CONFIG mode. Looking for ROOTConfig.cmake.
# Crucial for loading the proper definitions!
#
find_package(ROOT COMPONENTS CONFIG REQUIRED)
if(NOT ROOT_FOUND)
  message(STATUS "ROOT package not found.") 
  return()
endif()

message(STATUS "ROOT VERSION ${ROOT_VERSION}")

## Load macros: need to compile ROOT through cmake first and execute bin/thisroot.sh
include(${ROOT_USE_FILE})

#----------------------------------------------------------------------------
# Locate sources and headers for this project
# ## NOT NEEDED FOR DICT
include_directories(${PROJECT_SOURCE_DIR}/include 
                    ${PROJECT_SOURCE_DIR}/../shared/include 
                    ${Geant4_INCLUDE_DIR}
                    ${ROOT_INCLUDE_DIRS})

#----------------------------------------------------------------------------
# Add libraries: need to compile the Dict before linking WCSim !!
# in standard makefile, need to make rootcint anyway before standard make
#

ADD_CUSTOM_TARGET(LinkDirectories ALL
		COMMAND ${CMAKE_COMMAND} -E create_symlink ${CMAKE_CURRENT_SOURCE_DIR}/include 	${CMAKE_CURRENT_BINARY_DIR}/include
		COMMAND ${CMAKE_COMMAND} -E create_symlink ${CMAKE_CURRENT_SOURCE_DIR}/src 		${CMAKE_CURRENT_BINARY_DIR}/src)


## WCSimRootDict.cc regeneration by rootcint
## Use ROOT 5.34.32 as some issues with PARSE_ARGUMENTS were found in older ROOT versions (ROOT 5.34.11)
if ( ${ROOT_VERSION} GREATER_EQUAL 6 )
	# ROOT6 doesn't accept '/' in target and header names, use include_directories instead 
	# and move the dict source after
	include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
	ROOT_GENERATE_DICTIONARY(WCSimRootDict 
			WCSimRootEvent.hh 
			WCSimRootGeom.hh 
			WCSimPmtInfo.hh 
			WCSimEnumerations.hh 
			WCSimRootOptions.hh 
			TJNuBeamFlux.hh 
			TNRooTrackerVtx.hh 
			WCSimRootTools.hh 
			LINKDEF ${CMAKE_CURRENT_SOURCE_DIR}/include/WCSimRootLinkDef.hh)
	
	# Move ROOTDict in src directory
	ADD_CUSTOM_COMMAND(OUTPUT ./src/WCSimRootDict.cxx
			COMMAND ${CMAKE_COMMAND} -E copy WCSimRootDict.cxx 		${CMAKE_CURRENT_SOURCE_DIR}/src/WCSimRootDict.cxx 
			COMMAND ${CMAKE_COMMAND} -E remove WCSimRootDict.cxx
			MAIN_DEPENDENCY WCSimRootDict.cxx )
else()
	# ROOT5 need the full path
	ROOT_GENERATE_DICTIONARY(${CMAKE_CURRENT_SOURCE_DIR}/src/WCSimRootDict 
			${CMAKE_CURRENT_SOURCE_DIR}/include/WCSimRootEvent.hh 
			${CMAKE_CURRENT_SOURCE_DIR}/include/WCSimRootGeom.hh 
			${CMAKE_CURRENT_SOURCE_DIR}/include/WCSimPmtInfo.hh 
			${CMAKE_CURRENT_SOURCE_DIR}/include/WCSimEnumerations.hh 
			${CMAKE_CURRENT_SOURCE_DIR}/include/WCSimRootOptions.hh 
			${CMAKE_CURRENT_SOURCE_DIR}/include/TJNuBeamFlux.hh 
			${CMAKE_CURRENT_SOURCE_DIR}/include/TNRooTrackerVtx.hh 
			${CMAKE_CURRENT_SOURCE_DIR}/include/WCSimRootTools.hh 
			LINKDEF ${CMAKE_CURRENT_SOURCE_DIR}/include/WCSimRootLinkDef.hh)
			
endif()




## Crucial for reading ROOT classes: make shared object library
add_library(WCSimRoot SHARED 
		./src/WCSimRootEvent.cc 
		./src/WCSimRootGeom.cc 
		./src/WCSimPmtInfo.cc 
		./src/WCSimEnumerations.cc 
		./src/WCSimRootOptions.cc 
		./src/TJNuBeamFlux.cc 
		./src/TNRooTrackerVtx.cc 
		./src/WCSimRootTools.cc 
		./src/WCSimRootDict.cxx)
target_link_libraries(WCSimRoot  ${ROOT_LIBRARIES})

# Create libWCSimRootDict.so (needed for ROOT6)
add_custom_command(TARGET WCSimRoot
		POST_BUILD
		COMMAND ${CMAKE_COMMAND} -E create_symlink ${CMAKE_CURRENT_BINARY_DIR}/libWCSimRoot.so 	${CMAKE_CURRENT_BINARY_DIR}/libWCSimRootDict.so)

#----------------------------------------------------------------------------
# Find Geant4 package, activating all available UI and Vis drivers by default
# You can set WITH_GEANT4_UIVIS to OFF via the command line or ccmake/cmake-gui
# to build a batch mode only executable
#
option(WITH_GEANT4_UIVIS "Build example with Geant4 UI and Vis drivers" ON)
if(WITH_GEANT4_UIVIS)
  find_package(Geant4 REQUIRED ui_all vis_all)
else()
  find_package(Geant4 REQUIRED)
endif()

#----------------------------------------------------------------------------
# Setup Geant4 include directories and compile definitions
# Setup include directory for this project
#
include(${Geant4_USE_FILE})  ## NOT needed for Dict


#----------------------------------------------------------------------------
# Locate sources and headers for this project
# NB: headers are included so they will show up in IDEs
#
file(GLOB sources ${PROJECT_SOURCE_DIR}/src/*.cc)
file(GLOB headers ${PROJECT_SOURCE_DIR}/include/*.hh)


#----------------------------------------------------------------------------
# Add the executable, and link it to the Geant4 libraries
#

# To use <unordered_map>, ie. use C++11 now from GCC 4.4 onwards support for unordered_maps.
# Had to change G4float to G4double for QE because of double arithmatic inside the array.
if (CMAKE_COMPILER_IS_GNUCC AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 4.7)
    set(CMAKE_CXX_FLAGS " ${CMAKE_CXX_FLAGS} -std=c++11")
else()
    set(CMAKE_CXX_FLAGS " ${CMAKE_CXX_FLAGS} -std=c++0x")
endif()



add_executable(WCSim WCSim.cc ${sources} ${headers})
target_link_libraries(WCSim ${Geant4_LIBRARIES} ${ROOT_LIBRARIES} WCSimRoot Tree)  #add profiler to use gperftools



#----------------------------------------------------------------------------
# Copy all scripts to the build directory, i.e. the directory in which we
# build WCSim. This is so that we can run the executable directly because it
# relies on these scripts being in the current working directory.
#
set(WCSIM_SCRIPTS
  macros/jobOptions.mac
  WCSim.mac
  macros/daq.mac
  macros/visOGLSX.mac
  macros/visOGLQT.mac
  macros/visRayTracer.mac
  macros/mPMT_nuPrism1.mac
  macros/mPMT_nuPrism2.mac
  macros/tuning_parameters.mac
  macros/NuPRISM.mac
  macros/mPMT_interesting1.mac
  macros/mPMT_interesting2_SKwCover.mac
  macros/mPMT_37pmtHK.mac
  macros/mPMT_radon_with_bias.mac
  macros/mPMT_radon_without_bias.mac
  biasprofile.dat
  tuningNominal.mac
  WCSim_hybrid.mac
  WCSim_hybrid.sh
  mPMT-configfiles/mPMTconfig_19_nuPrism.txt
  mPMT-configfiles/mPMTconfig_30_13_3.txt
  mPMT-configfiles/mPMTconfig_33_50.txt
  mPMT-configfiles/mPMTconfig_34_22.5_1.txt
  mPMT-configfiles/mPMTconfig_34_22_1.txt
  mPMT-configfiles/mPMTconfig_37_HK.txt
  mPMT-configfiles/mPMTconfig_19_nuPrism_3ring.txt
  mPMT-configfiles/mPMTconfig_33_13_1.txt
  mPMT-configfiles/mPMTconfig_33_50_1.txt
  mPMT-configfiles/mPMTconfig_34_22.5_2.txt
  mPMT-configfiles/mPMTconfig_34_22_2.txt
  )

foreach(_script ${WCSIM_SCRIPTS})
  configure_file(
    ${PROJECT_SOURCE_DIR}/${_script}
    ${PROJECT_BINARY_DIR}/${_script}
    COPYONLY
    )
endforeach()

#----------------------------------------------------------------------------
# For internal Geant4 use - but has no effect if you build this
# example standalone
#
#add_custom_target(WCSim DEPENDS WCSim)

#----------------------------------------------------------------------------
# Install the executable to 'bin' directory under CMAKE_INSTALL_PREFIX
#
#install(TARGETS WCSim DESTINATION bin)

