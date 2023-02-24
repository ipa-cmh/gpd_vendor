cmake_minimum_required(VERSION 3.8 FATAL_ERROR)
project(gpd_vendor)
set(CMAKE_BUILD_TYPE Debug)
set(CMAKE_CXX_STANDARD 14)
set(PNAME gpd)

find_package(ament_cmake REQUIRED)
find_package(ament_cmake_ros REQUIRED)
find_package(Eigen3 REQUIRED)
find_package(VTK REQUIRED)
# PCL
find_package(PCL 1.9 REQUIRED COMPONENTS common visualization io filters surface keypoints features segmentation search kdtree octree registration)
#find_package(PCL 1.8 REQUIRED)

# OpenCV
find_package(OpenCV REQUIRED)

set(deps
  "Eigen3"
  "PCL"
  "OpenCV"
  "VTK"
)
#find_package(OpenCV REQUIRED PATHS /usr/local NO_DEFAULT_PATH)

## Set compiler optimization flags
# set(CMAKE_CXX_FLAGS "-O3 -fopenmp -fPIC -Wno-deprecated -Wenum-compare -std=c++14")
#set(CMAKE_CXX_FLAGS "-O3 -fopenmp -fPIC -Wno-deprecated -Wenum-compare -Wno-ignored-attributes -std=c++17")
#set(CMAKE_CXX_FLAGS "-fopenmp -fPIC -Wno-deprecated -Wenum-compare -Wno-ignored-attributes -std=c++17")
# set(CMAKE_CXX_FLAGS "-O3 -fopenmp -fPIC -Wno-deprecated -Wenum-compare")
# set(CMAKE_CXX_FLAGS "-O3 -fopenmp -march=native -mfpmath=sse -funroll-loops -fPIC -Wno-deprecated -Wenum-compare") # no improvement
# set(CMAKE_CXX_FLAGS "-frename-registers -Ofast -march=native -fopenmp -fPIC -Wno-deprecated -Wenum-compare") # no improvement

#set(CMAKE_CXX_FLAGS "-fopenmp -fPIC -Wno-deprecated -Wenum-compare -Wno-ignored-attributes -std=c++17")
set(CMAKE_CXX_FLAGS "-O3 -march=native -mtune=intel -msse4.2 -mavx2 -mfma -flto -fopenmp -fPIC -Wno-deprecated -Wenum-compare -Wno-ignored-attributes -std=c++17")

## Specify additional locations of header files
#include_directories(include ${PCL_INCLUDE_DIRS} ${OpenCV_INCLUDE_DIRS})

# Classifier depends on what libraries are installed.
option(USE_OPENVINO "use classifier based on OpenVINO toolkit" OFF)
option(USE_CAFFE "use classifier based on Caffe framework" OFF)
option(USE_OPENCV "use classifier based on OpenCV framework" OFF)
if(USE_OPENVINO STREQUAL "ON")
  find_package(InferenceEngine 1.0)
  if (NOT InferenceEngine_FOUND)
    message(FATAL_ERROR "Please install OpenVINO https://software.intel.com/en-us/articles/OpenVINO-Install-Linux")
  endif()
  include_directories(${InferenceEngine_INCLUDE_DIRS})
  link_directories(${InferenceEngine_LIBRARY_DIRS})
  add_definitions(-DUSE_OPENVINO)
  get_filename_component(MODELS_DIR "models/openvino" ABSOLUTE)
  configure_file(include/gpd/net/openvino_classifier.h.in gpd/net/openvino_classifier.h)
  include_directories(${CMAKE_CURRENT_BINARY_DIR})
  set(classifier_src src/${PNAME}/net/classifier.cpp src/${PNAME}/net/openvino_classifier.cpp)
  set(classifier_ament "InferenceEngine")
  message("Using OpenVINO")
elseif(USE_CAFFE STREQUAL "ON")
  find_package(Caffe)
  if (NOT Caffe_FOUND)
    message(FATAL_ERROR "Please install Caffe https://caffe.berkeleyvision.org/installation.html")
  endif()
  include_directories(${Caffe_INCLUDE_DIRS})
  add_definitions(${Caffe_DEFINITIONS})
  add_definitions(-DUSE_CAFFE)
  set(classifier_src src/${PNAME}/net/classifier.cpp src/${PNAME}/net/caffe_classifier.cpp)
  set(classifier_ament "Caffe" "OpenCV")
elseif(USE_OPENCV STREQUAL "ON")
  message(FATAL_ERROR "Not supported yet")
else()
  set(classifier_src 
    "src/${PNAME}/net/classifier.cpp"
    "src/${PNAME}/net/eigen_classifier.cpp"
    "src/${PNAME}/net/conv_layer.cpp"
    "src/${PNAME}/net/dense_layer.cpp")
endif()

# Optional PCL GPU operations
option(USE_PCL_GPU "use PCL GPU for point cloud operations" OFF)
if(USE_PCL_GPU STREQUAL "ON")
  FIND_PACKAGE(CUDA)
  if (NOT CUDA_FOUND)
    message(FATAL_ERROR "Please install CUDA and compile PCL with Nvidia GPU flags http://pointclouds.org/documentation/tutorials/gpu_install.php")
  endif()
  INCLUDE(FindCUDA)
  add_definitions(-DUSE_PCL_GPU)
  message("Using the GPU with PCL")
endif()

# Optional data generation (requires OpenCV Contrib)
option(BUILD_DATA_GENERATION "build data generation (requires OpenCV Contrib for HDF5)" OFF)
if(BUILD_DATA_GENERATION STREQUAL "ON")
  add_library(${PNAME}_data_generator STATIC src/${PNAME}/data_generator.cpp)
  target_link_libraries(${PNAME}_data_generator
   ${PNAME}_grasp_detector)
  ament_target_dependencies(${PNAME}_data_generator
    PCL
    OpenCV
    Eigen3)
  add_executable(${PNAME}_generate_data src/generate_data.cpp)
  target_link_libraries(${PNAME}_generate_data
   ${PNAME}_data_generator)
  message("Building data generation module")
endif()

# Generate the shared library from the sources
add_library(${PNAME}_grasp_detector SHARED 
  src/${PNAME}/grasp_detector.cpp
  src/${PNAME}/clustering.cpp
  src/${PNAME}/sequential_importance_sampling.cpp
  src/${PNAME}/candidate/antipodal.cpp
  src/${PNAME}/candidate/candidates_generator.cpp
  src/${PNAME}/candidate/finger_hand.cpp
  src/${PNAME}/candidate/frame_estimator.cpp
  src/${PNAME}/candidate/hand.cpp
  src/${PNAME}/candidate/hand_set.cpp
  src/${PNAME}/candidate/hand_geometry.cpp
  src/${PNAME}/candidate/hand_search.cpp
  src/${PNAME}/candidate/local_frame.cpp
  src/${PNAME}/util/cloud.cpp
  src/${PNAME}/util/config_file.cpp
  src/${PNAME}/util/eigen_utils.cpp
  src/${PNAME}/util/plot.cpp
  src/${PNAME}/util/point_list.cpp
  src/${PNAME}/descriptor/image_strategy.cpp
  src/${PNAME}/descriptor/image_1_channels_strategy.cpp
  src/${PNAME}/descriptor/image_3_channels_strategy.cpp
  src/${PNAME}/descriptor/image_12_channels_strategy.cpp
  src/${PNAME}/descriptor/image_15_channels_strategy.cpp
  src/${PNAME}/descriptor/image_geometry.cpp
  src/${PNAME}/descriptor/image_generator.cpp
  ${classifier_src}
)
target_include_directories(${PNAME}_grasp_detector PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
  ${PCL_INCLUDE_DIRS}
  ${OpenCV_INCLUDE_DIRS}
  ${Eigen3_INCLUDE_DIRS}
  )
target_compile_features(${PNAME}_grasp_detector PUBLIC c_std_99 cxx_std_14)
target_compile_options(${PNAME}_grasp_detector PUBLIC -fPIC -Wl,--no-undefined)
target_compile_definitions(${PNAME}_grasp_detector PUBLIC
  ${PCL_DEFINITIONS}
  ${OpenCV_DEFINITIONS}
  ${Eigen3_DEFINITIONS}
  )
target_link_libraries(${PNAME}_grasp_detector
  ${PCL_LIBRARIES}
  ${OpenCV_LIBS}
  ${Eigen3_LIBRARIES}
  )




## Declare C++ executables
add_executable(${PNAME}_cem_detect_grasps src/cem_detect_grasps.cpp)
target_include_directories(${PNAME}_cem_detect_grasps PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>)
target_compile_features(${PNAME}_cem_detect_grasps PUBLIC c_std_99 cxx_std_14)
target_compile_options(${PNAME}_cem_detect_grasps PUBLIC -fPIC -Wl,--no-undefined)
target_link_libraries(${PNAME}_cem_detect_grasps
  ${PNAME}_grasp_detector)
ament_target_dependencies(${PNAME}_cem_detect_grasps ${deps})

add_executable(${PNAME}_detect_grasps src/detect_grasps.cpp)
target_include_directories(${PNAME}_detect_grasps PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>)
target_compile_features(${PNAME}_detect_grasps PUBLIC c_std_99 cxx_std_14)
target_compile_options(${PNAME}_detect_grasps PUBLIC -fPIC -Wl,--no-undefined)
target_link_libraries(${PNAME}_detect_grasps
  ${PNAME}_grasp_detector)
ament_target_dependencies(${PNAME}_detect_grasps ${deps})

add_executable(${PNAME}_generate_candidates src/generate_candidates.cpp)
target_include_directories(${PNAME}_generate_candidates PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>)
target_compile_features(${PNAME}_generate_candidates PUBLIC c_std_99 cxx_std_14)
target_compile_options(${PNAME}_generate_candidates PUBLIC -fPIC -Wl,--no-undefined)
target_link_libraries(${PNAME}_generate_candidates
  ${PNAME}_grasp_detector)
ament_target_dependencies(${PNAME}_generate_candidates ${deps})

add_executable(${PNAME}_label_grasps src/label_grasps.cpp)
target_include_directories(${PNAME}_label_grasps PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>)
target_compile_features(${PNAME}_label_grasps PUBLIC c_std_99 cxx_std_14)
target_compile_options(${PNAME}_label_grasps PUBLIC -fPIC -Wl,--no-undefined)
target_link_libraries(${PNAME}_label_grasps
  ${PNAME}_grasp_detector)
ament_target_dependencies(${PNAME}_label_grasps ${deps})

add_executable(${PNAME}_test_grasp_image src/tests/test_grasp_image.cpp)
target_include_directories(${PNAME}_test_grasp_image PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>)
target_compile_features(${PNAME}_test_grasp_image PUBLIC c_std_99 cxx_std_14)
target_compile_options(${PNAME}_test_grasp_image PUBLIC -fPIC -Wl,--no-undefined)
target_link_libraries(${PNAME}_test_grasp_image
  ${PNAME}_grasp_detector)
ament_target_dependencies(${PNAME}_test_grasp_image ${deps})


# Set the location for library installation
install(TARGETS 
  ${PNAME}_grasp_detector
  EXPORT export_${PROJECT_NAME}
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin)

install(TARGETS ${PNAME}_test_grasp_image ${PNAME}_label_grasps ${PNAME}_generate_candidates ${PNAME}_detect_grasps ${PNAME}_cem_detect_grasps
  DESTINATION lib/${PROJECT_NAME}
)

install(DIRECTORY include/gpd DESTINATION include)

ament_export_include_directories(
  include
)
ament_export_libraries(
  ${PNAME}_grasp_detector
)

ament_export_targets(
  export_${PROJECT_NAME}
)

ament_export_dependencies(
  "Eigen3"
  "PCL"
  "OpenCV"
)

ament_package()