cmake_minimum_required(VERSION 2.8.11)

if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    add_definitions(-DVK_USE_PLATFORM_WIN32_KHR -DWIN32_LEAN_AND_MEAN)
    set(DisplayServer Win32)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Android")
    add_definitions(-DVK_USE_PLATFORM_ANDROID_KHR)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    if (BUILD_WSI_XCB_SUPPORT)
        add_definitions(-DVK_USE_PLATFORM_XCB_KHR)
    endif()

    if (BUILD_WSI_XLIB_SUPPORT)
       add_definitions(-DVK_USE_PLATFORM_XLIB_KHR)
    endif()

    if (BUILD_WSI_WAYLAND_SUPPORT)
       add_definitions(-DVK_USE_PLATFORM_WAYLAND_KHR)
    endif()

    if (BUILD_WSI_MIR_SUPPORT)
        add_definitions(-DVK_USE_PLATFORM_MIR_KHR)
        include_directories(${MIR_INCLUDE_DIR})
    endif()
else()
    message(FATAL_ERROR "Unsupported Platform!")
endif()

# On Windows, we must pair Debug and Release appropriately
if (WIN32)
   # For Windows, since 32-bit and 64-bit items can co-exist, we build each in its own build directory.
   # 32-bit target data goes in build32, and 64-bit target data goes into build.  So, include/link the
   # appropriate data at build time.
   if (CMAKE_CL_64)
      set (BUILDTGT_DIR build)
   else ()
      set (BUILDTGT_DIR build32)
   endif()
   file(COPY vk_layer_validation_tests.vcxproj.user DESTINATION ${CMAKE_BINARY_DIR}/tests)
endif()

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")

if(WIN32)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_CRT_SECURE_NO_WARNINGS -D_USE_MATH_DEFINES -DMAGICKCORE_HDRI_ENABLE=0 -DMAGICKCORE_QUANTUM_DEPTH=16")

    # If MSVC, disable some signed/unsigned mismatch warnings.
    if (MSVC)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4267")
    endif()

else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -DMAGICKCORE_HDRI_ENABLE=0 -DMAGICKCORE_QUANTUM_DEPTH=16")
endif()

set (LIBGLM_INCLUDE_DIR ${PROJECT_SOURCE_DIR}/libs)

set(COMMON_CPP
    vkrenderframework.cpp
    vktestbinding.cpp
    vktestframework.cpp
    test_environment.cpp
   )

include_directories(
    "${PROJECT_SOURCE_DIR}/tests"
    "${PROJECT_SOURCE_DIR}/tests/gtest-1.7.0/include"
    "${PROJECT_SOURCE_DIR}/icd/common"
    "${PROJECT_SOURCE_DIR}/layers"
    ${XCB_INCLUDE_DIRS}
    ${GLSLANG_SPIRV_INCLUDE_DIR}
    ${LIBGLM_INCLUDE_DIR}
    ${CMAKE_CURRENT_BINARY_DIR}
    ${CMAKE_BINARY_DIR}
    )

if (NOT WIN32)
    # extra setup for out-of-tree builds
    if (NOT (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR))
        add_custom_target(binary-dir-symlinks ALL
            COMMAND ln -sf ${CMAKE_CURRENT_SOURCE_DIR}/run_all_tests.sh
            COMMAND ln -sf ${CMAKE_CURRENT_SOURCE_DIR}/run_wrap_objects_tests.sh
            COMMAND ln -sf ${CMAKE_CURRENT_SOURCE_DIR}/run_loader_tests.sh
            COMMAND ln -sf ${CMAKE_CURRENT_SOURCE_DIR}/run_extra_loader_tests.sh
            COMMAND ln -sf ${CMAKE_CURRENT_SOURCE_DIR}/vkvalidatelayerdoc.sh
            # Files unique to VulkanTools go below this line
            COMMAND ln -sf ${CMAKE_CURRENT_SOURCE_DIR}/vktracereplay.sh
            VERBATIM
            )
    endif()
else()
    if (NOT (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR))
        FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/_run_all_tests.ps1 RUN_ALL)
        FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/_vktracereplay.ps1 VKTRACEREPLAY)
        FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/_vkvalidatelayerdoc.ps1 VKVALIDATELAYERDOC)
        FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/trace_positive_validation.ps1 VKTRACEPOSITIVE)
        add_custom_target(binary-dir-symlinks ALL
            COMMAND ${CMAKE_COMMAND} -E copy_if_different ${RUN_ALL} _run_all_tests.ps1
            COMMAND ${CMAKE_COMMAND} -E copy_if_different ${VKTRACEREPLAY} _vktracereplay.ps1
            COMMAND ${CMAKE_COMMAND} -E copy_if_different ${VKVALIDATELAYERDOC} _vkvalidatelayerdoc.ps1
            COMMAND ${CMAKE_COMMAND} -E copy_if_different ${VKTRACEPOSITIVE} trace_positive_validation.ps1
            VERBATIM
            )
    endif()
endif()

if(WIN32)
   set (LIBVK "${API_LOWERCASE}-${MAJOR}")
elseif(UNIX)
   set (LIBVK "${API_LOWERCASE}")
else()
endif()

add_executable(vk_layer_validation_tests layer_validation_tests.cpp ${COMMON_CPP})
set_target_properties(vk_layer_validation_tests
   PROPERTIES
   COMPILE_DEFINITIONS "GTEST_LINKED_AS_SHARED_LIBRARY=1")
if(NOT WIN32)
    if (BUILD_WSI_XCB_SUPPORT OR BUILD_WSI_XLIB_SUPPORT)
        target_link_libraries(vk_layer_validation_tests ${LIBVK} ${XCB_LIBRARIES} ${X11_LIBRARIES} gtest gtest_main VkLayer_utils ${GLSLANG_LIBRARIES})
    else()
        target_link_libraries(vk_layer_validation_tests ${LIBVK} gtest gtest_main VkLayer_utils ${GLSLANG_LIBRARIES})
    endif()
endif()
if(WIN32)
   target_link_libraries(vk_layer_validation_tests ${LIBVK} gtest gtest_main VkLayer_utils ${GLSLANG_LIBRARIES})
endif()
add_dependencies(vk_layer_validation_tests
   VkLayer_core_validation
   VkLayer_object_tracker
   VkLayer_threading
   VkLayer_unique_objects
   VkLayer_parameter_validation
)

if (WIN32)
    # For Windows, copy necessary gtest DLLs to the right spot for the vk_layer_tests...
    FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_BINARY_DIR}/gtest-1.7.0/$<CONFIGURATION>/*.dll SRC_GTEST_DLLS)
    FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_BINARY_DIR}/$<CONFIGURATION> DST_GTEST_DLLS)
    add_custom_command(TARGET vk_layer_validation_tests POST_BUILD
        COMMAND xcopy /Y /I ${SRC_GTEST_DLLS} ${DST_GTEST_DLLS})
endif()

add_executable(vk_loader_validation_tests loader_validation_tests.cpp ${COMMON_CPP})
set_target_properties(vk_loader_validation_tests
   PROPERTIES
   COMPILE_DEFINITIONS "GTEST_LINKED_AS_SHARED_LIBRARY=1")
target_link_libraries(vk_loader_validation_tests ${LIBVK} gtest gtest_main VkLayer_utils  ${GLSLANG_LIBRARIES})

add_subdirectory(gtest-1.7.0)
add_subdirectory(layers)
