
cmake_minimum_required(VERSION 2.8)
project(jetson-stream-lite)

# submodule warning
message(" ")
message("Note:  this project uses git submodules in the source tree.")
message("       if you haven't already, run the following command from")
message("       the project's root directory:")
message(" ")
message("           git submodule update --init") 
message("\n")

if( NOT EXISTS "${PROJECT_SOURCE_DIR}/utils/.git" )
	message("Note:  required git submodules have not been detected.")
	message("       first, please run the following command from the")
	message("       the project's root directory to clone them:")
	message(" ")
	message("          git submodule update --init")
	message(" ")
	message(FATAL_ERROR "missing required git submodules, see instructions above")
endif()


# setup build flags
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wno-write-strings -Wno-deprecated-declarations")	# -std=c++14 
set(BUILD_DEPS "YES" CACHE BOOL "If YES, will install dependencies into sandbox.  Automatically reset to NO after dependencies are installed.")
set(BUILD_INTERACTIVE "YES" CACHE BOOL "If NO, will download/install the default DNN models without prompting the user, and skip installation of PyTorch.")
set(BUILD_EXPERIMENTAL "NO" CACHE BOOL "If YES, will enable support for experimental DNNs, examples, and plugins")



# if this is the first time running cmake, perform pre-build dependency install script (or if the user manually triggers re-building the dependencies)
if( ${BUILD_DEPS} )
	message("-- Launching pre-build dependency installer script...")
	message("-- Build interactive:  ${BUILD_INTERACTIVE}")

	execute_process(COMMAND sh ../CMakePreBuild.sh ${BUILD_INTERACTIVE}
				WORKING_DIRECTORY ${PROJECT_BINARY_DIR}
				RESULT_VARIABLE PREBUILD_SCRIPT_RESULT)

	set(BUILD_DEPS "NO" CACHE BOOL "If YES, will install dependencies into sandbox.  Automatically reset to NO after dependencies are installed." FORCE)
	message("-- Finished installing dependencies")
endif()


# setup CUDA
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/utils/cuda" )
find_package(CUDA)
message("-- CUDA version: ${CUDA_VERSION}")

set(
	CUDA_NVCC_FLAGS
	${CUDA_NVCC_FLAGS}; 
    -O3 
	-gencode arch=compute_53,code=sm_53
	-gencode arch=compute_62,code=sm_62
)

if(CUDA_VERSION_MAJOR GREATER 9)
	message("-- CUDA ${CUDA_VERSION_MAJOR} detected, enabling SM_72")

	set(
		CUDA_NVCC_FLAGS
		${CUDA_NVCC_FLAGS}; 
		-gencode arch=compute_72,code=sm_72
	)
endif()

# setup project output paths
set(PROJECT_OUTPUT_DIR  ${PROJECT_BINARY_DIR}/${CMAKE_SYSTEM_PROCESSOR})
set(PROJECT_INCLUDE_DIR ${PROJECT_OUTPUT_DIR}/include)

file(MAKE_DIRECTORY ${PROJECT_INCLUDE_DIR})
file(MAKE_DIRECTORY ${PROJECT_OUTPUT_DIR}/bin)

message("-- system arch:  ${CMAKE_SYSTEM_PROCESSOR}")
message("-- output path:  ${PROJECT_OUTPUT_DIR}")

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_OUTPUT_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_OUTPUT_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_OUTPUT_DIR}/lib)


# build C/C++ library
include_directories(${PROJECT_INCLUDE_DIR} ${PROJECT_INCLUDE_DIR}/jetson-stream-lite ${PROJECT_INCLUDE_DIR}/jetson-utils)
include_directories(/usr/include/gstreamer-1.0 /usr/lib/aarch64-linux-gnu/gstreamer-1.0/include /usr/include/glib-2.0 /usr/include/libxml2 /usr/lib/aarch64-linux-gnu/glib-2.0/include/)

file(GLOB inferenceSources src/c/*.cpp src/c/*.cu src/calibration/*.cpp)
file(GLOB inferenceIncludes src/c/*.h src/c/*.cuh src/calibration/*.h)
file(GLOB pluginSources plugins/*.cpp plugins/*.cu)

cuda_add_library(jetson-stream-lite SHARED ${inferenceSources} ${pluginSources})


# transfer all headers to the include directory
file(MAKE_DIRECTORY ${PROJECT_INCLUDE_DIR}/jetson-stream-lite)

foreach(include ${inferenceIncludes})
	message("-- Copying ${include}")
	configure_file(${include} ${PROJECT_INCLUDE_DIR}/jetson-stream-lite COPYONLY)
endforeach()



# build subdirectories
add_subdirectory(utils)
add_subdirectory(app)


# set linker options
target_link_libraries(jetson-stream-lite jetson-utils nvinfer nvinfer_plugin nvcaffe_parser)

if(CUDA_VERSION_MAJOR GREATER 9)
	target_link_libraries(jetson-stream-lite nvonnxparser)

    if(HAS_OPENCV) 
        message("-- Linking jetson-stream-lite with OpenCV " ${OpenCV_VERSION})
        target_link_libraries(jetson-stream-lite opencv_core opencv_calib3d)
    endif()
endif()


# install includes
foreach(include ${inferenceIncludes})
    install(FILES "${include}" DESTINATION include/jetson-stream-lite)
endforeach()

# install symlink to networks and images
install(CODE "execute_process( COMMAND ${CMAKE_COMMAND} -E create_symlink ${PROJECT_SOURCE_DIR}/data/networks ${CMAKE_INSTALL_PREFIX}/bin/networks )" )
install(CODE "execute_process( COMMAND ${CMAKE_COMMAND} -E create_symlink ${PROJECT_SOURCE_DIR}/data/images ${CMAKE_INSTALL_PREFIX}/bin/images )" )

# install the shared library
install(TARGETS jetson-stream-lite DESTINATION lib EXPORT jetson-stream-liteConfig)

# install the cmake project, for importing
install(EXPORT jetson-stream-liteConfig DESTINATION share/jetson-stream-lite/cmake)

# run ldconfig after installing
install(CODE "execute_process( COMMAND ldconfig )")

