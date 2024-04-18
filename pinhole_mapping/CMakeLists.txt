SET(PROJECT_NAME pinhole_mapping)
CMAKE_MINIMUM_REQUIRED(VERSION 2.8.3)
PROJECT(${PROJECT_NAME})

#设置使用c++14
ADD_COMPILE_OPTIONS(-std=c++14 ) #添加了编译选项，指定了 C++ 标准为 C++14。
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fexceptions" ) #设置了 C 编译器的额外编译选项，启用了异常处理。
set(CMAKE_CXX_STANDARD 14) #设置了 C++ 标准为 C++14
set(CMAKE_CXX_STANDARD_REQUIRED ON) #设置了 C++ 标准的强制要求为开启状态。
set(CMAKE_CXX_EXTENSIONS OFF) #关闭了 C++ 扩展
# 设置了 C++ 编译器的额外编译选项，包括启用调试信息、优化级别为 O3、链接了一些库等。
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -std=c++14 -g -O3 -ltbb -lboost_system -msse2 -msse3 -pthread -w") # -Wall

SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}/CMakeModules/")
SET(CMAKE_BUILD_TYPE Release) # Release, RelWithDebInfo

find_package(CUDA REQUIRED)
SET(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS};-O3 -use_fast_math)

# Specify compute capability
list(APPEND CUDA_NVCC_FLAGS -arch=sm_61)
list(APPEND CUDA_NVCC_FLAGS -gencode arch=compute_61,code=sm_61)

SET(CUDA_PROPAGATE_HOST_FLAGS OFF)

# SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")


option(BUILD_ROS_NODE "Build ROS node" ON)
if(BUILD_ROS_NODE)
  ADD_DEFINITIONS(-DBUILD_ROS_NODE)
endif(BUILD_ROS_NODE)

option(BUILD_TESTS "Build test programs" ON)
if(BUILD_TESTS)
  ADD_DEFINITIONS(-DRMD_BUILD_TESTS)
endif(BUILD_TESTS)

# NCC patch size (must be odd integer, typical values: 3, 5, 7)
ADD_DEFINITIONS(-DRMD_CORR_PATCH_SIDE=5)
# Maximum extent of epipolar line search in pixels
ADD_DEFINITIONS(-DRMD_MAX_EXTENT_EPIPOLAR_SEARCH=100)

# include(/home/wang/software/opencv/opencv-3.2.0.bin/OpenCVConfig.cmake)
find_package(OpenCV REQUIRED)
find_package(Eigen REQUIRED)
find_package(Boost REQUIRED COMPONENTS system filesystem)


if(BUILD_ROS_NODE)
  find_package(catkin REQUIRED COMPONENTS
    roscpp roslib cmake_modules cv_bridge image_transport pcl_ros sensor_msgs geometry_msgs)
  catkin_package(
      DEPENDS OpenCV Eigen Boost
      CATKIN_DEPENDS roscpp roslib image_transport pcl_ros
      INCLUDE_DIRS include
      LIBRARIES pinhole_mapping pinhole_mapping_cuda
  )
else()
  set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
  set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)
endif(BUILD_ROS_NODE)

include_directories(
  include
  ${OpenCV_INCLUDE_DIRS}
  ${Eigen_INCLUDE_DIRS}
  ${Boost_INCLUDE_DIRS}
)

# Set link libraries
list(APPEND LINK_LIBS
  ${OpenCV_LIBS}
  ${Boost_LIBRARIES}
)

if(BUILD_ROS_NODE)
  include_directories(${catkin_INCLUDE_DIRS})
  list(APPEND LINK_LIBS ${catkin_LIBRARIES})
endif(BUILD_ROS_NODE)

# Set CUDA sourcefiles
list(APPEND CU_SOURCE_FILES
    src/check_cuda_device.cu
    src/seed_matrix.cu
)

# Set sourcefiles
list(APPEND SOURCE_FILES
    src/depthmap.cpp
)

CUDA_ADD_LIBRARY(
    pinhole_mapping_cuda
    ${CU_SOURCE_FILES}
)

target_link_libraries(
    pinhole_mapping_cuda
    ${LINK_LIBS}
)

ADD_LIBRARY(
    pinhole_mapping
    ${SOURCE_FILES}
)

target_link_libraries(
    pinhole_mapping
    ${LINK_LIBS}
)

# Build the ROS node
add_executable(
  pinhole_mapping_node
  src/depthmap_node.cpp
  src/publisher.cpp
  src/main_ros.cpp
)

target_link_libraries(
  pinhole_mapping_node
  pinhole_mapping
  pinhole_mapping_cuda
  ${OpenCV_LIBS}
)