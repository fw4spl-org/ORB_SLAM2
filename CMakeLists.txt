cmake_minimum_required(VERSION 3.1)
project(ORB_SLAM2)

SET(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
SET(CMAKE_SKIP_BUILD_RPATH FALSE)
SET(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE) 
SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

message("Build type: " ${CMAKE_BUILD_TYPE})

if(UNIX)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-rpath ${CMAKE_INSTALL_RPATH}")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}  -Wall  -O3 -march=native")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall   -O3 -march=native")

    # Check C++11 or C++0x support
    include(CheckCXXCompilerFlag)

    CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
    CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
    if(COMPILER_SUPPORTS_CXX11)
       set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
       add_definitions(-DCOMPILEDWITHC11)
       message(STATUS "Using flag -std=c++11.")
    elseif(COMPILER_SUPPORTS_CXX0X)
       set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
       add_definitions(-DCOMPILEDWITHC0X)
       message(STATUS "Using flag -std=c++0x.")
    else()
       message(FATAL_ERROR "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
    endif()
else()
    set(CMAKE_CXX_STANDARD 11)
    set(CMAKE_CXX_STANDARD_REQUIRED ON)
    add_definitions(-DCOMPILEDWITHC11)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /bigobj")
endif()

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake_modules)

find_package(OpenCV REQUIRED)
find_package(Eigen3 3.1.0 REQUIRED)

# DBoW2 target
add_subdirectory(${PROJECT_SOURCE_DIR}/Thirdparty/DBoW2/)
# g2o target
add_subdirectory(${PROJECT_SOURCE_DIR}/Thirdparty/g2o/)

add_library(ORB_SLAM2 SHARED
    src/System.cc
    src/Tracking.cc
    src/LocalMapping.cc
    src/LoopClosing.cc
    src/ORBextractor.cc
    src/ORBmatcher.cc
    src/Converter.cc
    src/MapPoint.cc
    src/KeyFrame.cc
    src/Map.cc
    src/Optimizer.cc
    src/PnPsolver.cc
    src/Frame.cc
    src/KeyFrameDatabase.cc
    src/Sim3Solver.cc
    src/Initializer.cc
)

target_include_directories(ORB_SLAM2 PUBLIC
    ${PROJECT_SOURCE_DIR}
    ${PROJECT_SOURCE_DIR}/Thirdparty/DBoW2
    ${PROJECT_SOURCE_DIR}/Thirdparty/g2o
    ${PROJECT_SOURCE_DIR}/include
    ${OpenCV_INCLUDE_DIRS}
    ${EIGEN3_INCLUDE_DIR}
    ${PROJECT_BINARY_DIR}
)

target_link_libraries(ORB_SLAM2
    ${OpenCV_LIBS}
    ${EIGEN3_LIBS}
    DBoW2
    g2o
)

include(GenerateExportHeader)
set_target_properties(ORB_SLAM2 PROPERTIES CXX_VISIBILITY_PRESET hidden)
set_target_properties(ORB_SLAM2 PROPERTIES VISIBILITY_INLINES_HIDDEN 1)
generate_export_header(ORB_SLAM2)

install(TARGETS ORB_SLAM2
  RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/bin
  ARCHIVE DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
  LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib )
install(DIRECTORY ${PROJECT_SOURCE_DIR}/include/
  DESTINATION ${CMAKE_INSTALL_PREFIX}/include/ORB_SLAM2/
  FILES_MATCHING PATTERN "*.h")
install(FILES ${PROJECT_BINARY_DIR}/orb_slam2_export.h
  DESTINATION ${CMAKE_INSTALL_PREFIX}/include/ORB_SLAM2/)

message(STATUS "Compile With map save/load function")
find_library(BOOST_SERIALIZATION boost_serialization)
if (NOT BOOST_SERIALIZATION)
    message(FATAL_ERROR "Can't find libboost_serialization")
endif()
target_link_libraries(${PROJECT_NAME} ${BOOST_SERIALIZATION})

if(WIN32)
    add_definitions(
        -DBOOST_ALL_NO_LIB
    )
endif()

# Build examples
option(BUILD_EXAMPLES "Build ORB_SLAM2 examples." OFF)
if(BUILD_EXAMPLES)
    add_executable(rgbd_tum
    Examples/RGB-D/rgbd_tum.cc)
    target_link_libraries(rgbd_tum ORB_SLAM2)

    add_executable(stereo_kitti
    Examples/Stereo/stereo_kitti.cc)
    target_link_libraries(stereo_kitti ORB_SLAM2)

    add_executable(stereo_euroc
    Examples/Stereo/stereo_euroc.cc)
    target_link_libraries(stereo_euroc ${PROJECT_NAME})

    add_executable(mono_tum
    Examples/Monocular/mono_tum.cc)
    target_link_libraries(mono_tum ORB_SLAM2)

    add_executable(mono_kitti
    Examples/Monocular/mono_kitti.cc)
    target_link_libraries(mono_kitti ORB_SLAM2)

    # Install examples
    install(TARGETS
        rgbd_tum
        stereo_kitti
        mono_tum
        mono_kitti
        RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/bin
    )
endif()

# Build utility
option(BUILD_UTILITY "Build ORB_SLAM2 utility." OFF)
if(BUILD_UTILITY)
    add_executable(bin_vocabulary Vocabulary/bin_vocabulary.cpp)

    target_include_directories(bin_vocabulary PUBLIC
        ${PROJECT_SOURCE_DIR}
        ${PROJECT_SOURCE_DIR}/include
        ${PROJECT_SOURCE_DIR}/Thirdparty/DBoW2
        ${OpenCV_INCLUDE_DIRS}
        ${PROJECT_BINARY_DIR}/Thirdparty/DBoW2
    )
    target_link_libraries(bin_vocabulary DBoW2 ${OpenCV_LIBS})
    # Install utility
    install(TARGETS bin_vocabulary RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)
endif()

# Install text vocabulary file
set(EXECUTE_COMMAND_TAR ${CMAKE_COMMAND} -E tar xfz ${PROJECT_SOURCE_DIR}/Vocabulary/ORBvoc.txt.tar.gz)
install(CODE "execute_process(COMMAND ${EXECUTE_COMMAND_TAR} WORKING_DIRECTORY ${CMAKE_INSTALL_PREFIX}/bin)")
