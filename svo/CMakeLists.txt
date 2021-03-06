################################################################################
# user build settings

SET(TRACE TRUE)
SET(HAVE_G2O TRUE) 

################################################################################

SET(PROJECT_NAME svo)
PROJECT(${PROJECT_NAME})
CMAKE_MINIMUM_REQUIRED (VERSION 2.8.3)
SET(CMAKE_BUILD_TYPE Release) # Release, RelWithDebInfo
SET(CMAKE_VERBOSE_MAKEFILE OFF)
SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}/CMakeModules/")

# Set definitions
IF(TRACE)
  ADD_DEFINITIONS(-DSVO_TRACE)
ENDIF()
IF(HAVE_G2O)
  ADD_DEFINITIONS(-DUSE_BUNDLE_ADJUSTMENT)
ENDIF()
ADD_DEFINITIONS(-DSVO_USE_ROS)

# Set build flags, set ARM_ARCHITECTURE environment variable on Odroid
SET(CMAKE_CXX_FLAGS "-Wall -Werror -D_LINUX -D_REENTRANT -march=native -Wno-unused-variable -Wno-unused-but-set-variable -Wno-unknown-pragmas")
IF(DEFINED ENV{ARM_ARCHITECTURE})
  SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS} -O3 -mfpu=neon -march=armv7-a -fomit-frame-pointer -fsee -funsafe-loop-optimizations -fno-signed-zeros -fno-math-errno -funroll-loops")
ELSE()
  SET(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS} -O3 -mmmx -msse -msse -msse2 -msse3 -mssse3 -fomit-frame-pointer -fsee -funsafe-loop-optimizations -fno-signed-zeros -fno-math-errno -funroll-loops")
ENDIF()
IF(CMAKE_COMPILER_IS_GNUCC)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
ELSE()
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
ENDIF()

# Add catkin and required ROS packages 
FIND_PACKAGE(catkin REQUIRED COMPONENTS roscpp roslib cmake_modules vikit_common vikit_ros)

# Add plain cmake packages 
FIND_PACKAGE(OpenCV REQUIRED)
FIND_PACKAGE(Eigen REQUIRED)
FIND_PACKAGE(Sophus REQUIRED)
FIND_PACKAGE(fast REQUIRED)
FIND_PACKAGE(Boost REQUIRED COMPONENTS thread)

# Describe catkin Project
catkin_package(
    DEPENDS Eigen OpenCV Sophus Boost fast
    CATKIN_DEPENDS roscpp roslib vikit_common vikit_ros
    INCLUDE_DIRS include
    LIBRARIES svo
)

# Include dirs
INCLUDE_DIRECTORIES(
  include
  ${Eigen_INCLUDE_DIRS}
  ${OpenCV_INCLUDE_DIRS}
  ${Sophus_INCLUDE_DIRS}
  ${Boost_INCLUDE_DIRS}
  ${catkin_INCLUDE_DIRS} 
  ${fast_INCLUDE_DIRS}
)

# Set link libraries
LIST(APPEND LINK_LIBS 
  ${OpenCV_LIBRARIES}
  ${Sophus_LIBRARIES}
  ${Boost_LIBRARIES}
  ${catkin_LIBRARIES}
  ${fast_LIBRARIES}
)

# Set sourcefiles
LIST(APPEND SOURCEFILES
  src/frame_handler_mono.cpp
  src/frame_handler_base.cpp
  src/frame.cpp
  src/point.cpp
  src/map.cpp
  src/pose_optimizer.cpp
  src/initialization.cpp
  src/matcher.cpp
  src/reprojector.cpp
  src/feature_alignment.cpp
  src/feature_detection.cpp
  src/depth_filter.cpp
  src/config.cpp
  src/sparse_img_align.cpp)

# Add g2o if available
IF(HAVE_G2O)
  FIND_PACKAGE(G2O REQUIRED)
  INCLUDE_DIRECTORIES(
    ${G2O_ROOT}/include
    /usr/include/suitesparse # for cholmod
  )
  LIST(APPEND LINK_LIBS
    g2o_core g2o_stuff g2o_solver_cholmod g2o_solver_csparse g2o_solver_dense
    g2o_solver_pcg g2o_types_sba cholmod cxsparse
  )
  LIST(APPEND SOURCEFILES src/bundle_adjustment.cpp)
ENDIF(HAVE_G2O)

# Create svo library
ADD_LIBRARY(svo SHARED ${SOURCEFILES})
TARGET_LINK_LIBRARIES(svo ${LINK_LIBS})


################################################################################
# TESTS

ADD_EXECUTABLE(test_feature_align test/test_feature_alignment.cpp)
TARGET_LINK_LIBRARIES(test_feature_align svo)

ADD_EXECUTABLE(test_pipeline test/test_pipeline.cpp)
TARGET_LINK_LIBRARIES(test_pipeline svo)

ADD_EXECUTABLE(test_matcher test/test_matcher.cpp)
TARGET_LINK_LIBRARIES(test_matcher svo)

ADD_EXECUTABLE(test_feature_detection test/test_feature_detection.cpp)
TARGET_LINK_LIBRARIES(test_feature_detection svo)

ADD_EXECUTABLE(test_depth_filter test/test_depth_filter.cpp)
TARGET_LINK_LIBRARIES(test_depth_filter svo)

ADD_EXECUTABLE(test_sparse_img_align test/test_sparse_img_align.cpp)
TARGET_LINK_LIBRARIES(test_sparse_img_align svo)

ADD_EXECUTABLE(test_pose_optimizer test/test_pose_optimizer.cpp)
TARGET_LINK_LIBRARIES(test_pose_optimizer svo) 
