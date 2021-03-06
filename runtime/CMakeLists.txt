CMAKE_MINIMUM_REQUIRED(VERSION 2.8)

add_definitions("-std=gnu++11")

option(CUDA_VERBOSE_PTXAS
  "On to enable verbose output from the PTXAS assembler."
  OFF
  )
  
option(CUDA_DEVICE_DEBUGGING
  "On to enable debugging device code."
  OFF
  )  

option(USE_MPI
  "On to build support for MPI"
  ON
  )

option(USE_GALACTICS
  "Include John Dubinsky galatics IC generator"
  OFF
  )

option(USE_GALACTICS_IFORT
  "Set to ON if galactics is compipled with ifort"
  OFF
)


option(USE_OPENGL
  "On to build support for OpenGL Rendering"
  OFF
  )

option(USE_THRUST
  "On to use thrust for sorting"
  OFF
  )

option(USE_B40C
  "On to use B40C sorting"
  ON
  )
  
option(COMPILE_SM30
  "On to enable SM30 compilation"
  ON
  )

option(USE_DUST
  "On to enable dust rendering"
  OFF
  )
 
option(CUDA_KEEP_INTERMEDIATE_FILES
  "On to enable -keep"
  OFF
  )

option(WAR_OF_GALAXIES
  "On to enable WarOfGalaxies"
  OFF
  )

FIND_PACKAGE(CUDA REQUIRED)

if (USE_MPI)
  add_definitions(-DUSE_MPI)
endif (USE_MPI)

if (USE_THRUST)
  add_definitions(-DUSE_THRUST)
  set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -DTHRUST_DEBUG")
#  include_directories(${CUDA_TOOLKIT_ROOT_DIR}/../../thrust)
endif (USE_THRUST)

if (USE_B40C)
  add_definitions(-DUSE_B40C)
endif (USE_B40C)

if (USE_DUST)
  add_definitions(-DUSE_DUST)
  set(BINARY_NAME bonsai2)
else (USE_DUST)
  set(BINARY_NAME bonsai2_slowdust)
endif (USE_DUST)

set(CMAKE_DEBUG_POSTFIX "D")

include_directories(${CMAKE_SOURCE_DIR}/include ${CMAKE_SOURCE_DIR}/renderer)

set (CCFILES
  src/build.cpp 
  src/compute_properties.cpp 
  src/gpu_iterate.cpp 
  src/libraryInterface.cpp
  src/load_kernels.cpp
  src/main.cpp
  src/anyoption.cpp
  src/octree.cpp
  src/parallel.cpp
  src/sort_bodies_gpu.cpp
  src/dustFunctions.cpp
  src/log.cpp
  src/hostConstruction.cpp
  src/Galaxy.cpp
  src/FileIO.cpp
  src/WOGManager.cpp
)

set (HFILES
  include/my_cuda_rt.h
  include/my_ocl.h
  include/node_specs.h
  include/octree.h
  include/build.h
  include/tipsydefs.h
  include/vector_math.h
  include/depthSort.h
  include/sort.h
)

set (CUFILES
  CUDAkernels/build_tree.cu
  CUDAkernels/compute_propertiesD.cu
  CUDAkernels/particles.cu
  CUDAkernels/sortKernels.cu
  CUDAkernels/timestep.cu
  CUDAkernels/depthSort.cu
  CUDAkernels/dev_direct_gravity.cu
  CUDAkernels/war_of_galaxies.cu
)

if (COMPILE_SM30)
  set (CUFILES 
    ${CUFILES}
    CUDAkernels/dev_approximate_gravity_warp_new.cu  
    )
  add_definitions("-DKEPLER")
  set(GENCODE -gencode arch=compute_35,code=sm_35 -gencode arch=compute_35,code=compute_35 -Xptxas=-v -ftz=true)

else (COMPILE_SM30)
  set (CUFILES
    ${CUFILES}
	CUDAkernels/dev_approximate_gravity_warp_fermi.cu  
	)
  set(GENCODE -gencode arch=compute_20,code=sm_20 -gencode arch=compute_20,code=compute_20)
endif (COMPILE_SM30)

set (CUHFILES
  CUDAkernels/support_kernels.cu
  CUDAkernels/scanKernels.cu
  )

set (PROFFILES
  # profiling/cuxTimer.cu
  # profiling/cuxTimer_host.cu
  profiling/cuxTimer.h
  profiling/cuxTimer_private.h
  profiling/warp_functions.h
  profiling/derived_atomic_functions.h
  profiling/bonsai_timing.h
  )

source_group("CUDA Source Files" FILES ${CUFILES} ${CUHFILES})
source_group("CUDA Include Files" FILES ${CUHFILES})
source_group("CUDA Profiling Tools" FILES ${PROFFILES})

if (CUDA_KEEP_INTERMEDIATE_FILES)
  set(KEEP -keep)
endif(CUDA_KEEP_INTERMEDIATE_FILES)

if (CUDA_VERBOSE_PTXAS)
  set(VERBOSE_PTXAS --ptxas-options=-v)
endif (CUDA_VERBOSE_PTXAS)

if (CUDA_DEVICE_DEBUGGING)
  set(DEVICE_DEBUGGING -G)
endif (CUDA_DEVICE_DEBUGGING)

if (WIN32)
  add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
endif (WIN32)

add_definitions(-DTIPSYOUTPUT)

if (USE_OPENGL) 
  set(CCFILES ${CCFILES} src/renderloop.cpp src/render_particles.cpp)
  set(HFILES ${HFILES} include/renderloop.h include/render_particles.h)
  
  set (CCFILES ${CCFILES} 
  renderer/SmokeRenderer.cpp
  renderer/SmokeShaders.cpp
  renderer/paramgl.cpp
  renderer/param.cpp
  renderer/GLSLProgram.cpp
  renderer/framebufferObject.cpp
  renderer/loadPPM.cpp
  renderer/Cubemap.cpp  
  src/tr.c
  )

  set (HFILES ${HFILES} 
  renderer/SmokeRenderer.h
  renderer/SmokeShaders.h
  renderer/paramgl.h
  renderer/param.h
  renderer/GLSLProgram.h
  renderer/framebufferObject.h
  renderer/loadPPM.h
  renderer/Cubemap.h
  include/tr.h
  )

  add_definitions(-DUSE_OPENGL)
  FIND_PACKAGE(OpenGL REQUIRED)

  # Link the right freeglut
  if (WIN32)
    if( CMAKE_SIZEOF_VOID_P EQUAL 8 )
      set( LIB_PATH ${CMAKE_SOURCE_DIR}/lib/win64/ )
	  set( GLEW_NAME glew64 )
    else( CMAKE_SIZEOF_VOID_P EQUAL 8 )
	  set( LIB_PATH ${CMAKE_SOURCE_DIR}/lib/win32/ )
	  set( GLEW_NAME glew32 )
    endif( CMAKE_SIZEOF_VOID_P EQUAL 8 )
	else (WIN32)
		set( LIB_PATH ${CMAKE_SOURCE_DIR}/lib/linux )
	  set( GLEW_NAME GLEW_x86_64 )
  endif (WIN32)

  #freeglut
  FIND_PATH(FREEGLUT_INCLUDE_DIR NAMES GL/freeglut.h PATHS ${CMAKE_SOURCE_DIR}/include)
	if (WIN32)
		FIND_LIBRARY(FREEGLUT_LIBRARY NAMES freeglut PATHS ${LIB_PATH})
	else (WIN32)
		FIND_LIBRARY(FREEGLUT_LIBRARY NAMES glut PATHS ${LIB_PATH})
	endif (WIN32)
  

  #glew
  FIND_PATH(GLEW_INCUDE_DIR NAMES GL/glew.h PATHS ${CMAKE_SOURCE_DIR}/include)
	if (WIN32)
		FIND_LIBRARY(GLEW_LIBRARY NAMES ${GLEW_NAME} PATHS ${LIB_PATH})
	else (WIN32)
		FIND_LIBRARY(GLEW_LIBRARY NAMES ${GLEW_NAME} PATHS ${LIB_PATH})
	endif (WIN32)
  
  #glu
  if (WIN32)
    set(ALL_LIBRARIES ${FREEGLUT_LIBRARY} ${GLEW_LIBRARY})
  else (WIN32)
    FIND_LIBRARY(GLU_LIBRARY NAMES GLU PATHS ${LIB_PATH})
    set(ALL_LIBRARIES ${FREEGLUT_LIBRARY} ${GLEW_LIBRARY} ${GLU_LIBRARY} ${OPENGL_gl_LIBRARY} ${OPENGL_glu_LIBRARY})
  endif (WIN32)

  # Copy DLLs
	if (WIN32)
		file(COPY ${LIB_PATH}/freeglut.dll DESTINATION ${CMAKE_BINARY_DIR}) 
		file(COPY ${LIB_PATH}/${GLEW_NAME}.dll DESTINATION ${CMAKE_BINARY_DIR}) 
	endif (WIN32)

endif (USE_OPENGL)

if (WAR_OF_GALAXIES)
  add_definitions(-DWAR_OF_GALAXIES)
endif (WAR_OF_GALAXIES)

#The SSE support for the local tree walks
if(WIN32)
  add_definitions( "/arch:SSE4")
 else (WIN32)
   #  add_definitions( "-msse4 -std=gnu++0x")
     add_definitions( "-msse4")
endif (WIN32)

if (USE_MPI)
	#The OpenMP Library and compiler flags
	FIND_PACKAGE(OpenMP REQUIRED)
	if(OPENMP_FOUND)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
		set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
	endif()
endif(USE_MPI)

cuda_add_executable(${BINARY_NAME}
  ${CCFILES} 
  ${HFILES}
  ${CUFILES}
  ${PROFFILES}
  OPTIONS ${GENCODE} ${VERBOSE_PTXAS} ${DEVICE_DEBUGGING} ${KEEP}
  )

if (USE_GALACTICS)
  add_definitions("-DGALACTICS")
  if (USE_GALACTICS_IFORT)
    target_link_libraries(${BINARY_NAME} ${ALL_LIBRARIES} -L./ -lgengalaxy -lifcore)
  else(USE_GALACTICS_IFORT)
    target_link_libraries(${BINARY_NAME} ${ALL_LIBRARIES} -L./ -lgengalaxy -lgfortran)
  endif(USE_GALACTICS_IFORT)
else(USE_GALACTICS)
  target_link_libraries(${BINARY_NAME} ${ALL_LIBRARIES})
endif(USE_GALACTICS)

#copy test data file
file(COPY ${CMAKE_SOURCE_DIR}/model3_child_compact.tipsy DESTINATION ${CMAKE_BINARY_DIR})
file(COPY ${CMAKE_SOURCE_DIR}/../images/ DESTINATION ${CMAKE_BINARY_DIR}/../images/)

