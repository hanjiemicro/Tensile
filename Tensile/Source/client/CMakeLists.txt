
#set(CMAKE_CXX_COMPILER ${HIP_HIPCC_EXECUTABLE})

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14" )

set(client_sources
    source/BenchmarkTimer.cpp
    source/CSVStackFile.cpp
    source/ClientProblemFactory.cpp
    source/ConvolutionProblem.cpp
    source/DataInitialization.cpp
    source/HardwareMonitor.cpp
    source/HardwareMonitorListener.cpp
    source/MetaRunListener.cpp
    source/PerformanceReporter.cpp
    source/ProgressListener.cpp
    source/Reference.cpp
    source/ReferenceValidator.cpp
    source/ResultFileReporter.cpp
    source/ResultReporter.cpp
    source/SolutionIterator.cpp
    source/TimingEvents.cpp
    )

find_package(Boost  COMPONENTS program_options filesystem REQUIRED)

add_library(TensileClient STATIC ${client_sources})

find_package(ROCmSMI QUIET)
if(NOT ROCmSMI_FOUND)
    set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH}" "${Tensile_DIR}" "${Tensile_DIR}/../Source/cmake" "${CMAKE_HOME_DIRECTORY}/cmake")
    find_package(ROCmSMI REQUIRED)
endif()

target_include_directories(TensileClient PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/include")# "${rocm_smi_root}/include")

if(CXX_VERSION_STRING MATCHES "clang")
    target_compile_options(TensileClient PUBLIC -mno-xnack)
    set(gfx906_options "-Xarch_gfx906 -msram-ecc")
    set(gfx908_options "-Xarch_gfx908 -msram-ecc")
    if(CMAKE_VERSION VERSION_LESS 3.12)
        set_target_properties(TensileClient PROPERTIES
          COMPILE_FLAGS "${gfx906_options} ${gfx908_options}")
    else()
	    target_compile_options(TensileClient PUBLIC
          "SHELL:${gfx906_options}" "SHELL:${gfx908_options}")
    endif()
endif()

target_link_libraries(TensileClient TensileHost ${Boost_LIBRARIES} rocm_smi)

add_executable(tensile_client main.cpp)

target_link_libraries(tensile_client PRIVATE TensileHost TensileClient ${Boost_LIBRARIES})
if(OpenMP_FOUND)
    target_link_libraries(tensile_client PRIVATE "${OpenMP_EXE_LINKER_FLAGS}")
else()
    set(OPENMP_LIBRARY /usr/lib/x86_64-linux-gnu/libomp.so)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OPENMP_LIBRARY}")
    target_link_libraries(tensile_client PRIVATE "${OpenMP_EXE_LINKER_FLAGS}")
endif()

foreach(arch IN LISTS TENSILE_GPU_ARCHS)
    target_link_libraries(tensile_client PRIVATE "--amdgpu-target=${arch}")
endforeach(arch)
