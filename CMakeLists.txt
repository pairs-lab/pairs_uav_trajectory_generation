cmake_minimum_required(VERSION 3.5)
project(pairs_uav_trajectory_generation)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_definitions(-Wall)
add_definitions(-Wextra)

if(NOT PAIRS_ENABLE_TESTING)
  add_definitions(-O3)
endif()

if(COVERAGE)
  message(WARNING "building with --coverage, the performance might be limited")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} --coverage")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --coverage")
endif()

set(CATKIN_DEPENDENCIES
  cmake_modules
  roscpp
  rospy
  nodelet
  dynamic_reconfigure
  std_msgs
  pairs_msgs
  pairs_lib
  )

find_package(catkin REQUIRED COMPONENTS
  ${CATKIN_DEPENDENCIES}
  )

# Use the SYSTEM nlopt (libnlopt-cxx-dev), not the catkin ros-<distro>-nlopt
# component. The latter ships a header and library from mismatched nlopt versions
# (same SONAME libnlopt_cxx.so.0), which makes nlopt::opt::set_min_objective jump
# to the wrong symbol and segfault during trajectory optimization. Locate the
# system nlopt explicitly (NO_DEFAULT_PATH skips /opt/ros's broken copy); linking
# the system C library libnlopt.so.0 also avoids the SONAME collision, since
# ros-<distro>-nlopt only provides libnlopt_cxx.so.0. Matches the ROS2/jazzy build.
find_library(NLOPT_LIBRARY NAMES nlopt PATHS /usr/lib /usr/lib/x86_64-linux-gnu NO_DEFAULT_PATH)
find_path(NLOPT_INCLUDE_DIR NAMES nlopt.hpp PATHS /usr/include NO_DEFAULT_PATH)
if(NOT NLOPT_LIBRARY OR NOT NLOPT_INCLUDE_DIR)
  message(FATAL_ERROR "system nlopt not found — install libnlopt-cxx-dev")
endif()
message(STATUS "Using system nlopt: ${NLOPT_LIBRARY} (header ${NLOPT_INCLUDE_DIR})")

generate_dynamic_reconfigure_options(
  config/drs.cfg
  )

find_package(Eigen3 REQUIRED)
set(Eigen_INCLUDE_DIRS ${EIGEN3_INCLUDE_DIRS})
set(Eigen_LIBRARIES ${Eigen_LIBRARIES})

set(LIBRARIES
  PairsUavTrajectoryGeneration_EthTrajectoryGeneration
  PairsUavTrajectoryGeneration_PairsTrajectoryGeneration
  PairsUavTrajectoryGeneration_PathRandomFlier
  )

catkin_package(
  INCLUDE_DIRS include test/include
  LIBRARIES ${LIBRARIES}
  CATKIN_DEPENDS ${CATKIN_DEPENDENCIES}
  DEPENDS Eigen
  )

add_library(PairsUavTrajectoryGeneration_EthTrajectoryGeneration
  src/eth_trajectory_generation/motion_defines.cpp
  src/eth_trajectory_generation/polynomial.cpp
  src/eth_trajectory_generation/segment.cpp
  src/eth_trajectory_generation/timing.cpp
  src/eth_trajectory_generation/trajectory.cpp
  src/eth_trajectory_generation/trajectory_sampling.cpp
  src/eth_trajectory_generation/vertex.cpp
  src/eth_trajectory_generation/rpoly/rpoly_ak1.cpp
  )

add_library(PairsUavTrajectoryGeneration_PairsTrajectoryGeneration
  src/pairs_trajectory_generation.cpp
  )

# ensure dynamic_reconfigure drsConfig.h is generated before this lib compiles
# (upstream omits this; it only matters in isolated single-job bloom builds)
add_dependencies(PairsUavTrajectoryGeneration_PairsTrajectoryGeneration
  ${PROJECT_NAME}_gencfg
  ${${PROJECT_NAME}_EXPORTED_TARGETS}
  ${catkin_EXPORTED_TARGETS}
  )

include_directories(
  include
  test/include
  ${catkin_INCLUDE_DIRS}
  ${Eigen_INCLUDE_DIRS}
  ${NLOPT_INCLUDE_DIR}
  ${dynamic_reconfigure_PACKAGE_PATH}/cmake/cfgbuild.cmake
  )

target_link_libraries(PairsUavTrajectoryGeneration_PairsTrajectoryGeneration
  PairsUavTrajectoryGeneration_EthTrajectoryGeneration
  ${catkin_LIBRARIES}
  ${Eigen_LIBRARIES}
  ${NLOPT_LIBRARY}
  )

## | -------------------- path random flier ------------------- |

add_library(PairsUavTrajectoryGeneration_PathRandomFlier
  src/path_random_flier.cpp
  )

add_dependencies(PairsUavTrajectoryGeneration_PathRandomFlier
  ${${PROJECT_NAME}_EXPORTED_TARGETS}
  ${catkin_EXPORTED_TARGETS}
  )

target_link_libraries(PairsUavTrajectoryGeneration_PathRandomFlier
  ${catkin_LIBRARIES}
  )

## --------------------------------------------------------------
## |                           Testing                          |
## --------------------------------------------------------------

if(CATKIN_ENABLE_TESTING AND PAIRS_ENABLE_TESTING)

  message(WARNING "Testing enabled.")

  add_subdirectory(test)

endif()

## --------------------------------------------------------------
## |                           Install                          |
## --------------------------------------------------------------

install(TARGETS ${LIBRARIES}
  ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  RUNTIME DESTINATION ${CATKIN_GLOBAL_BIN_DESTINATION}
  )

install(DIRECTORY launch config
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}
  )

install(DIRECTORY scripts/
  USE_SOURCE_PERMISSIONS
  DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
  )

install(DIRECTORY scripts
  USE_SOURCE_PERMISSIONS
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}
  )

install(FILES nodelets.xml
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}
  )
