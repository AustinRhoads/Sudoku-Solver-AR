CMAKE_MINIMUM_REQUIRED(VERSION 3.6.2)

PROJECT(sudoku_solver_ar)

IF(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
	SET(LINUX True)
	SET(CMAKE_CXX_FLAGS_RELEASE " -O2 -s")
	SET(USE_AVX true CACHE BOOL "Use AVX functions")
	SET(USE_CUDA true CACHE BOOL "Build CUDA neural network trainer")
ELSEIF(WIN32)
	SET(GLFW_INCLUDE_DIR "" CACHE PATH "GLFW include directory")
	SET(GLFW_LIBRARY_DIR "" CACHE PATH "GLFW library dictory")
	SET(GLES3_INCLUDE_DIR "" CACHE PATH "OpenGL ES 3 include directory")
	SET(GLEW_INCLUDE_DIR "" CACHE PATH "GLEW include directory")
	SET(GLEW_LIBRARY_DIR "" CACHE PATH "GLEW library directory")
	SET(GLM_INCLUDE_DIR "" CACHE PATH "GLM include directory")
	LINK_DIRECTORIES("${GLFW_LIBRARY_DIR}" "${GLEW_LIBRARY_DIR}")
	SET(GUI_TYPE WIN32)
ENDIF()

FIND_PACKAGE(Freetype REQUIRED)
INCLUDE_DIRECTORIES("${FREETYPE_INCLUDE_DIRS}")

FIND_PACKAGE(OpenMP)

IF("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
	SET(EXTRA_CXX_FLAGS "-march=native") # GCC only because it makes performance about 2x slower on Clang.
ENDIF()

FIND_PROGRAM(CCACHE_FOUND ccache)
IF(CCACHE_FOUND)
	SET_PROPERTY(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ${CCACHE})
	SET_PROPERTY(GLOBAL PROPERTY RULE_LAUNCH_LINK ${CCACHE})
ENDIF()

SET(SOURCE_FILES
	src/sudoku_solver_ar.cpp
	src/CachedPuzzleSolver.cpp
	src/Camera.cpp
	src/DeltaTimer.cpp
	src/Game.cpp
	src/Geometry.cpp
	src/ImageProcessing.cpp
	src/NeuralNetwork.cpp
	src/NeuralNetworkData.cpp
	src/Painter.cpp
	src/PuzzleFinder.cpp
	src/ShaderProgram.cpp
	src/Solve.cpp
)

ADD_EXECUTABLE(sudoku_solver_ar ${GUI_TYPE} ${SOURCE_FILES})
IF(LINUX)
	SET_TARGET_PROPERTIES(sudoku_solver_ar PROPERTIES COMPILE_FLAGS "-Wall -Wtype-limits -Woverloaded-virtual -msse2 -msse3 -msse4.2 -mavx -std=c++1z ${EXTRA_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
	TARGET_LINK_LIBRARIES(sudoku_solver_ar glfw GL ${FREETYPE_LIBRARIES} gomp pthread)
	ADD_CUSTOM_TARGET(run DEPENDS sudoku_solver_ar WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}" COMMAND "${CMAKE_BINARY_DIR}/sudoku_solver_ar" || true)
	ADD_CUSTOM_TARGET(debug DEPENDS sudoku_solver_ar WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}" COMMAND "gdb" "${CMAKE_BINARY_DIR}/sudoku_solver_ar" "-ex" "run" || true)
ELSEIF(WIN32)
	ADD_DEFINITIONS(-DNOMINMAX -DGLEW_STATIC -D_USE_MATH_DEFINES)
	SET_TARGET_PROPERTIES(sudoku_solver_ar PROPERTIES COMPILE_FLAGS "/std:c++latest ${OpenMP_CXX_FLAGS}")
	SET_TARGET_PROPERTIES(sudoku_solver_ar PROPERTIES VS_DEBUGGER_WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}")
	TARGET_LINK_LIBRARIES(sudoku_solver_ar glfw3 opengl32 glew32s mfplat mfreadwrite mf mfuuid ${FREETYPE_LIBRARIES})
	TARGET_INCLUDE_DIRECTORIES(sudoku_solver_ar PRIVATE "${GLFW_INCLUDE_DIR}" "${GLES3_INCLUDE_DIR}" "${GLEW_INCLUDE_DIR}" "${GLM_INCLUDE_DIR}")
ENDIF()

IF(USE_AVX)
	ADD_DEFINITIONS(-DUSE_AVX)
ENDIF()

IF(LINUX)
	ADD_EXECUTABLE(sudoku_solver src/sudoku_solver.cpp src/Game.cpp src/Solve.cpp)
	SET_TARGET_PROPERTIES(sudoku_solver PROPERTIES COMPILE_FLAGS "-Wall -Wtype-limits -Woverloaded-virtual -msse2 -msse3 -std=c++1z")
ENDIF()

IF(USE_CUDA)
	FIND_PACKAGE(CUDA QUIET REQUIRED)
	SET(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS}; -std=c++11 -O3 -gencode arch=compute_30,code=sm_30 -lineinfo) # Tuned for GTX 6xx.
	CUDA_ADD_EXECUTABLE(train_neural_network
		src/train_neural_network.cu
		src/NeuralNetworkData.cpp
	)
ENDIF()
