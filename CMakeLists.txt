cmake_minimum_required(VERSION 3.10)

cmake_policy(PUSH)

# Path to the CC folder in CloudCompare, to require as little modification to this CMake as possible
set(CC_PATH_STR "../CloudCompare/CC" CACHE STRING "Path to CloudCompare folder")
get_filename_component(_CC_PATH ${CC_PATH_STR} ABSOLUTE)


# Options
option( COMPILE_CC_CORE_LIB_WITH_QT "Check to compile CC_CORE_LIB with Qt (to enable parallel processing)" ON )
option( COMPILE_CC_CORE_LIB_WITH_CGAL "Check to compile CC_CORE_LIB with CGAL lib. (to enable Delaunay 2.5D triangulation with a GPL compliant licence)" OFF )
option( COMPILE_CC_CORE_LIB_WITH_TBB " Check to compile CC_CORE_LIB with Intel Threading Building Blocks lib (enables some parallel processing )" ON )
option( COMPILE_CC_CORE_LIB_SHARED "Check to compile CC_CORE_LIB as a shared library (DLL/so)" ON )
set( OPTION_SCALAR_DOUBLE OFF CACHE BOOL "Check to define ScalarType as double"  )


# to compile CCLib only! (CMake implicitly imposes to declare a project before anything...)
project( CC_CORE_LIB VERSION 1.0 )

if ( OPTION_SCALAR_DOUBLE )
  add_definitions( -DSCALAR_TYPE_DOUBLE )
else()
  add_definitions( -DSCALAR_TYPE_FLOAT )
endif()


set(INSTALL_DESTINATIONS ${CMAKE_INSTALL_PREFIX}/bin )

include ( ${_CC_PATH}/../cmake/CMakePolicies.cmake )
include ( ${_CC_PATH}/../cmake/CMakeSetCompilerOptions.cmake )


if (COMPILE_CC_CORE_LIB_WITH_CGAL)
	include( ${_CC_PATH}/cmake/CGALSupport.cmake )
endif()

if (COMPILE_CC_CORE_LIB_WITH_TBB)
	include( ${_CC_PATH}/cmake/FindTBB.cmake )
else()
	if (NOT WIN32)
		message( WARNING "CCLib configured without parallel algorithm capabilities - see COMPILE_CC_CORE_LIB_WITH_TBB" )
	endif()
endif()

# Additional dependencies (only Qt in fact)
if (COMPILE_CC_CORE_LIB_WITH_QT)
	include( ${_CC_PATH}/../cmake/CMakeExternalLibs.cmake )
endif()


file( GLOB header_list ${_CC_PATH}/include/*.h)
file( GLOB source_list ${_CC_PATH}/src/*.cpp src/*.h)


if ( WIN32 )
	# Add Qt5 natvis file
	if ( COMPILE_CC_CORE_LIB_WITH_QT )
		list( APPEND header_list ${_CC_PATH}/../scripts/windows/qt5.natvis )
		#message(${header_list})
	endif()
endif()

if (COMPILE_CC_CORE_LIB_SHARED)
	set( CMAKE_CXX_VISIBILITY_PRESET hidden )
	set( CMAKE_VISIBILITY_INLINES_HIDDEN 1 )
	
	add_library( ${PROJECT_NAME} SHARED ${header_list} ${source_list} )
else()
	add_library( ${PROJECT_NAME} STATIC ${header_list} ${source_list} )
	
	target_compile_definitions( ${PROJECT_NAME} PRIVATE CC_CORE_LIB_STATIC_DEFINE )	
endif()

target_include_directories(CC_CORE_LIB 
	PUBLIC
		$<BUILD_INTERFACE:${_CC_PATH}/include>
    $<INSTALL_INTERFACE:include>
	 )

# generate the export header file
include( GenerateExportHeader )

generate_export_header( ${PROJECT_NAME}
	EXPORT_FILE_NAME ${CMAKE_CURRENT_BINARY_DIR}/CCCoreLibExport.h 
	EXPORT_MACRO_NAME CC_CORE_LIB_API
)

list( APPEND header_list ${CMAKE_CURRENT_BINARY_DIR}/CCCoreLibExport.h )

if (COMPILE_CC_CORE_LIB_WITH_CGAL)
	target_link_libraries( ${PROJECT_NAME} ${CGAL_LIBRARIES} )
	set_property( TARGET ${PROJECT_NAME} APPEND PROPERTY COMPILE_DEFINITIONS USE_CGAL_LIB )
endif()

if (COMPILE_CC_CORE_LIB_WITH_TBB)
	target_include_directories(${PROJECT_NAME} PUBLIC ${TBB_INCLUDE_DIRS} )
	target_link_libraries( ${PROJECT_NAME} PUBLIC ${TBB_LIBRARIES} )
	set_property( TARGET ${PROJECT_NAME} APPEND PROPERTY COMPILE_DEFINITIONS USE_TBB )
endif()

if ( COMPILE_CC_CORE_LIB_WITH_QT )
	target_include_directories(${PROJECT_NAME} PUBLIC ${Qt5Widgets_INCLUDE_DIRS} ${Qt5Core_INCLUDE_DIRS} )
	target_link_libraries(${PROJECT_NAME} PUBLIC Qt5::Core Qt5::Widgets Qt5::Concurrent)
	set_property( TARGET ${PROJECT_NAME} APPEND PROPERTY COMPILE_DEFINITIONS USE_QT )
endif()

# Load advanced scripts
include( ${_CC_PATH}/../cmake/CMakeInclude.cmake )

if ( COMPILE_CC_CORE_LIB_SHARED )
	# Install (shared) library to specified destinations
	if( WIN32 OR APPLE )
		foreach( dest ${INSTALL_DESTINATIONS} )
			install_shared( CC_CORE_LIB ${dest} 1 )
		endforeach()
	else()
		#install_shared( CC_CORE_LIB ${CMAKE_INSTALL_LIBDIR}/cloudcompare 0 ) #default destination: /usr/lib/cloudcompare
		set_target_properties(CC_CORE_LIB PROPERTIES PUBLIC_HEADER "${header_list}")

		install(TARGETS CC_CORE_LIB
        EXPORT CC_CORE_LIBTargets
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib
        RUNTIME DESTINATION bin
        INCLUDES DESTINATION include
        PUBLIC_HEADER DESTINATION include/cloudcompare
        )

		include( GNUInstallDirs )		

    install(EXPORT CC_CORE_LIBTargets
        FILE CC_CORE_LIBTargets.cmake
        NAMESPACE CC_CORE_LIB::
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/CC_CORE_LIB  
    )

    
    include(CMakePackageConfigHelpers)
    configure_package_config_file(CC_CORE_LIBConfig.cmake.in
				${CMAKE_CURRENT_BINARY_DIR}/CC_CORE_LIBConfig.cmake					
				INSTALL_DESTINATION ${LIB_INSTALL_DIR}/CC_CORE_LIB/cmake  		
				PATH_VARS CMAKE_INSTALL_LIBDIR  
		)

		write_basic_package_version_file(
	    	CC_CORE_LIBConfigVersion.cmake
	    	VERSION ${PACKAGE_VERSION}
	    	COMPATIBILITY AnyNewerVersion
    )    

    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/CC_CORE_LIBConfig.cmake"
              "${CMAKE_CURRENT_BINARY_DIR}/CC_CORE_LIBConfigVersion.cmake"
        DESTINATION lib/cmake/CC_CORE_LIB
        )

    include(CPackConfig.cmake)
   	include(CPack)


	endif()
endif()

if (WIN32)
	set_property( TARGET ${PROJECT_NAME} APPEND PROPERTY COMPILE_DEFINITIONS _CRT_SECURE_NO_WARNINGS )
endif()

set( CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -DCC_DEBUG" )
set( CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -DCC_DEBUG" )

cmake_policy(POP)
