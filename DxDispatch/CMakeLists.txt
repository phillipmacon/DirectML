cmake_minimum_required(VERSION 3.18)
project(dxdispatch VERSION 0.15.1 LANGUAGES CXX)

# ==============================================================================
# External Libraries/Helpers
# ==============================================================================

include(FetchContent)
include(cmake/helper_platform.cmake)

if(NOT TARGET_XBOX)
    # Statically link runtime library to avoid runtime dependency on Visual C++ redistributable.
    # On Xbox we deploy these dependencies for now.
    set(CMAKE_MSVC_RUNTIME_LIBRARY MultiThreaded$<$<CONFIG:Debug>:Debug>)
endif()

if(TARGET_WSL)
    set(CMAKE_SKIP_BUILD_RPATH FALSE)
    set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
    set(CMAKE_INSTALL_RPATH "$\{ORIGIN\}")
endif()

include(cmake/gsl.cmake)
include(cmake/rapidjson.cmake)
include(cmake/fmt.cmake)
include(cmake/wil.cmake)
include(cmake/half.cmake)
include(cmake/cxxopts.cmake)

if(NOT TARGET_XBOX)
    include(cmake/gtest.cmake)
endif()

include(cmake/wil.cmake)
add_wil_target(wil CACHE_PREFIX DXD)

include(cmake/gdk.cmake)
add_gdk_target(gdk CACHE_PREFIX DXD)
get_target_property(gdk_dxcompiler_path gdk DX_COMPILER_PATH)

include(cmake/pix.cmake)
add_pix_target(pix CACHE_PREFIX DXD)

include(cmake/dxcompiler.cmake)
add_dxcompiler_target(dxcompiler CACHE_PREFIX DXD GDK_DXCOMPILER_PATH ${gdk_dxcompiler_path})
get_target_property(dxcompiler_type dxcompiler DX_COMPONENT_CONFIG)

include(cmake/d3d12.cmake)
add_d3d12_target(d3d12 CACHE_PREFIX DXD)

include(cmake/directml.cmake)
add_directml_target(directml CACHE_PREFIX DXD)

include(cmake/onnxruntime.cmake)
add_onnxruntime_target(onnxruntime CACHE_PREFIX DXD)
get_target_property(onnxruntime_type onnxruntime DX_COMPONENT_CONFIG)

# ==============================================================================
# Model Library
# ==============================================================================
add_library(
    model STATIC 
    src/model/JsonParsers.cpp 
    src/model/JsonParsers.h
    src/model/Model.cpp
    src/model/Model.h
    src/model/NpyReaderWriter.cpp
    src/model/NpyReaderWriter.h
)

target_link_libraries(
    model 
    PRIVATE 
    fmt::fmt-header-only
    PUBLIC
    Microsoft.GSL::GSL
    Half::Half
    rapidjson::rapidjson
    wil
    d3d12
    directml
)

target_compile_features(model PRIVATE cxx_std_17)
target_precompile_headers(model PRIVATE src/model/pch.h)
target_include_directories(model INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/src/model)

# ==============================================================================
# Main Executable
# ==============================================================================
get_target_property(directml_config directml DX_COMPONENT_CONFIG)
get_target_property(d3d12_config d3d12 DX_COMPONENT_CONFIG)
get_target_property(dxcompiler_config dxcompiler DX_COMPONENT_CONFIG)
get_target_property(pix_config pix DX_COMPONENT_CONFIG)
get_target_property(gdk_config gdk DX_COMPONENT_CONFIG)
get_target_property(ort_config onnxruntime DX_COMPONENT_CONFIG)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/src/dxdispatch/config.h.in config.h)

add_executable(
    dxdispatch
    src/dxdispatch/Adapter.cpp
    src/dxdispatch/Adapter.h
    src/dxdispatch/Device.cpp
    src/dxdispatch/Device.h
    src/dxdispatch/main.cpp
    src/dxdispatch/DmlDispatchable.cpp
    src/dxdispatch/DmlDispatchable.h
    src/dxdispatch/Executor.cpp
    src/dxdispatch/Executor.h
    src/dxdispatch/CommandLineArgs.cpp
    src/dxdispatch/CommandLineArgs.h
    src/dxdispatch/Logging.cpp
    src/dxdispatch/Logging.h
    src/dxdispatch/PixCaptureHelper.cpp
    src/dxdispatch/PixCaptureHelper.h
    src/dxdispatch/DxModules.cpp
    src/dxdispatch/DxModules.h
    src/dxdispatch/ModuleInfo.cpp
    ${CMAKE_CURRENT_BINARY_DIR}/config.h
)

if(NOT dxcompiler_type STREQUAL None)
    target_sources(dxdispatch PRIVATE src/dxdispatch/HlslDispatchable.cpp)
endif()

if(NOT onnxruntime_type STREQUAL None)
    target_sources(dxdispatch PRIVATE src/dxdispatch/OnnxDispatchable.cpp)
endif()

set_property(DIRECTORY PROPERTY VS_STARTUP_PROJECT dxdispatch)

if(WIN32)
    target_sources(dxdispatch PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/dxdispatch.rc)
    target_link_libraries(dxdispatch PRIVATE version.lib)
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/src/dxdispatch/dxdispatch.rc.in dxdispatch.rc)
endif()

if(TARGET_WSL)
    target_link_libraries(dxdispatch PRIVATE -ldl)
endif()

target_include_directories(dxdispatch PRIVATE ${CMAKE_CURRENT_BINARY_DIR})

target_link_libraries(
    dxdispatch 
    PRIVATE 
    Microsoft.GSL::GSL
    fmt::fmt-header-only
    model
    cxxopts
    directml
    d3d12
    dxcompiler
    pix
    gdk
    wil
    onnxruntime
)

target_compile_features(dxdispatch PRIVATE cxx_std_17)
target_precompile_headers(dxdispatch PRIVATE src/dxdispatch/pch.h)
target_copy_redist_dependencies(dxdispatch)

install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/models DESTINATION bin)
install(
    FILES 
    $<TARGET_FILE:dxdispatch> 
    ${CMAKE_CURRENT_SOURCE_DIR}/ThirdPartyNotices.txt
    ${CMAKE_CURRENT_SOURCE_DIR}/doc/Guide.md
    DESTINATION bin
)

if(WIN32)
    install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/tools/AnalyzePixGpuCapture.ps1 DESTINATION bin/tools)
endif()

if(TARGET_XBOX)
    # Deploy to the console instead of running on local machine.
    set_property(TARGET dxdispatch PROPERTY VS_SOLUTION_DEPLOY ON)

    # No need for logos/assets; this is a developer-only app.
    target_sources(dxdispatch PRIVATE ${CMAKE_BINARY_DIR}/MicrosoftGame.config )
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/src/dxdispatch/dxdispatch.config.in MicrosoftGame.config)
    set_source_files_properties(${CMAKE_BINARY_DIR}/MicrosoftGame.config PROPERTIES VS_TOOL_OVERRIDE "MGCCompile")

    # Copy MSVC/UCRT redist files matching the current toolset to the build output directory. Not all of the DLLs
    # are necessary but this approach is simple and should avoid missing dependencies.
    set(CMAKE_INSTALL_SYSTEM_RUNTIME_LIBS_SKIP TRUE)
    # set(CMAKE_INSTALL_DEBUG_LIBRARIES TRUE)
    # set(CMAKE_INSTALL_DEBUG_LIBRARIES_ONLY TRUE)
    # set(CMAKE_INSTALL_UCRT_LIBRARIES TRUE)
    include(InstallRequiredSystemLibraries)
    add_custom_command(
        TARGET dxdispatch 
        POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_INSTALL_SYSTEM_RUNTIME_LIBS} $<TARGET_FILE_DIR:dxdispatch>
    )

    # For Copy models to the deployment directory even if other targets don't need to be built.
    add_custom_target(
        copy_models ALL
        COMMAND ${CMAKE_COMMAND} -E copy_directory "${CMAKE_CURRENT_SOURCE_DIR}/models" "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_VS_PLATFORM_NAME}/Layout/Image/Loose/models"
    )
endif()

# ==============================================================================
# Tests
# ==============================================================================
if((TARGET_WINDOWS OR TARGET_WSL) AND (TARGET_ARCH MATCHES "^X64|X86$"))
    option(DXD_TESTS "Build DxDispatch tests" ON)
else()
    option(DXD_TESTS "Build DxDispatch tests" OFF)
endif()

if(DXD_TESTS)
    enable_testing()
    include(GoogleTest)

    add_executable(
        jsontests 
        src/test/JsonParserTests.cpp
    )

    target_compile_features(jsontests PRIVATE cxx_std_17)
    target_link_libraries(
        jsontests 
        PRIVATE 
        gtest_main 
        fmt::fmt-header-only
        directml
        d3d12
        wil
        model
    )
    target_include_directories(jsontests PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src/dxdispatch)
    gtest_discover_tests(jsontests)

    # Hacky. Needed for silly reasons related to defining GUIDs in winadapter. This should
    # ideally be cleaned up at some point.
    if(NOT WIN32)
        add_dependencies(jsontests dxdispatch)
    endif()

    function(model_test model_name expected_output)
        add_test(NAME test_${model_name} COMMAND dxdispatch models/${model_name}.json WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
        set_tests_properties(test_${model_name} PROPERTIES PASS_REGULAR_EXPRESSION ${expected_output})
    endfunction()

    model_test(dml_convolution "Resource 'output': 6, 8, 12, 14")
    model_test(dml_cumulative_product "Resource 'Out': 2, 8, 64, 192")
    model_test(dml_element_wise_add "Resource 'Out': 6, 10, -2")
    model_test(dml_element_wise_add_npy "Resource 'Out': 2, 4, 6, 8, 10, 12")
    model_test(dml_element_wise_add1 "Resource 'Out': 6, 10, -0.432332")
    model_test(dml_element_wise_clip "Resource 'A': -2.5, -2.5, -2, -1, 0, 1, 2, 2.5, 2.5")
    model_test(dml_element_wise_identity "Resource 'Out': 12, 14, 16")
    model_test(dml_fill_value_sequence "Resource 'Out': 3.2, 4.7, 6.2, 7.7, 9.2")
    model_test(dml_join "Resource 'Out': 1, 2, 3, 100, 115, 5, 6, 7, 8, 9")
    model_test(dml_reduce "Resource 'output': 6, 15, 24")
    model_test(dml_slice "Resource 'output': 7, 9, 12, 14, 17, 19")
    model_test(dml_split "Resource 'Out1': 1, 2\nResource 'Out2': 3, 4\nResource 'Out3': 5, 6")
    model_test(dml_upsample_2d "Resource 'output': 1, 1.25, 1.75, 2, 1.5, 1.75, 2.25, 2.5, 2.5, 2.75, 3.25, 3.5, 3, 3.25, 3.75, 4")
    model_test(dml_owned_tensors "Resource 'Out': 6, 10, -2")
    if(NOT dxcompiler_type STREQUAL None)
        model_test(hlsl_add_fp32 "Resource 'Out': 2, 7, 6, 11, 2, 7")
    endif()
    if(NOT onnxruntime_type STREQUAL None)
        model_test(onnx_gemm "Resource 'Out': 22, 28, 34, 40, 46, 60, 74, 88, 70, 92, 114, 136")
    endif()
endif()