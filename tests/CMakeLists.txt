cmake_minimum_required(VERSION 2.8.11)

if(TARGET gtest_main)
    if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
        add_definitions(-DVK_USE_PLATFORM_WIN32_KHR -DWIN32_LEAN_AND_MEAN)
        # Workaround for TR1 deprecation in Visual Studio 15.5 until Google Test is updated
        add_definitions(-D_SILENCE_TR1_NAMESPACE_DEPRECATION_WARNING)
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
    elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
        add_definitions(-DVK_USE_PLATFORM_MACOS_MVK)
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
    endif()

    set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")

    if(WIN32)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_CRT_SECURE_NO_WARNINGS -D_USE_MATH_DEFINES")

        # If MSVC, disable some signed/unsigned mismatch warnings.
        if (MSVC)
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4267")
        endif()

    else()
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
    endif()

    set (LIBGLM_INCLUDE_DIR ${PROJECT_SOURCE_DIR}/libs)

    include_directories(
        ${PROJECT_SOURCE_DIR}/external
        ${GTEST_SOURCE_DIR}/googletest/include
        ${CMAKE_CURRENT_BINARY_DIR}
        ${CMAKE_BINARY_DIR}
        ${PROJECT_BINARY_DIR}
    )

    if (NOT WIN32)
        # extra setup for out-of-tree builds
        if (NOT (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR))
            add_custom_target(binary-dir-symlinks ALL
                COMMAND ln -sf ${CMAKE_CURRENT_SOURCE_DIR}/run_wrap_objects_tests.sh
                COMMAND ln -sf ${CMAKE_CURRENT_SOURCE_DIR}/run_loader_tests.sh
                COMMAND ln -sf ${CMAKE_CURRENT_SOURCE_DIR}/run_extra_loader_tests.sh
                COMMAND ln -sf ${CMAKE_CURRENT_SOURCE_DIR}/run_all_tests.sh
                VERBATIM
            )
        endif()
    else()
        if (NOT (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR))
            FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/_run_all_tests.ps1 RUN_ALL)
            add_custom_target(binary-dir-symlinks ALL
                COMMAND ${CMAKE_COMMAND} -E copy_if_different ${RUN_ALL} run_all_tests.ps1
                VERBATIM
            )
            set_target_properties(binary-dir-symlinks PROPERTIES FOLDER ${LOADER_HELPER_FOLDER})
        endif()
    endif()

    add_executable(vk_loader_validation_tests loader_validation_tests.cpp)
    set_target_properties(vk_loader_validation_tests
        PROPERTIES
        COMPILE_DEFINITIONS "GTEST_LINKED_AS_SHARED_LIBRARY=1")
    if(NOT WIN32)
        set_target_properties(vk_loader_validation_tests
            PROPERTIES
            COMPILE_FLAGS "-Wno-sign-compare")
    endif()

    if(TARGET vulkan)
        set(LOADER_LIB vulkan)
    elseif(WIN32 AND NOT $ENV{VULKAN_SDK} EQUAL "")
        if(CMAKE_SIZEOF_VOID_P EQUAL 8)
            set(LOADER_LIB "$ENV{VULKAN_SDK}/Lib/vulkan-1.lib")
        else()
            set(LOADER_LIB "$ENV{VULKAN_SDK}/Lib32/vulkan-1.lib")
        endif()
    else()
        set(LOADER_LIB vulkan)
    endif()

    target_link_libraries(vk_loader_validation_tests "${LOADER_LIB}" gtest gtest_main)
    if(BUILD_LOADER AND ENABLE_STATIC_LOADER)
        set_target_properties(vk_loader_validation_tests PROPERTIES LINK_FLAGS "/ignore:4098")
    endif()
    if(WIN32)
        file(COPY vk_loader_validation_tests.vcxproj.user DESTINATION "${CMAKE_CURRENT_BINARY_DIR}")
    endif()

    # Copy loader and googletest (gtest) libs to test dir so the test executable can find them.
    if(WIN32)
        if (CMAKE_GENERATOR MATCHES "^Visual Studio.*")
            file(TO_NATIVE_PATH ${PROJECT_BINARY_DIR}/external/googletest/googletest/$<CONFIG>/gtest_main$<$<CONFIG:Debug>:d>.dll GTEST_COPY_SRC1)
            file(TO_NATIVE_PATH ${PROJECT_BINARY_DIR}/external/googletest/googletest/$<CONFIG>/gtest$<$<CONFIG:Debug>:d>.dll GTEST_COPY_SRC2)
            file(TO_NATIVE_PATH ${CMAKE_CURRENT_BINARY_DIR}/$<CONFIG> GTEST_COPY_DEST)
        else()
            file(TO_NATIVE_PATH ${PROJECT_BINARY_DIR}/external/googletest/googeltest/gtest_main.dll GTEST_COPY_SRC1)
            file(TO_NATIVE_PATH ${PROJECT_BINARY_DIR}/external/googletest/googletest/gtest.dll GTEST_COPY_SRC2)
            file(TO_NATIVE_PATH ${CMAKE_CURRENT_BINARY_DIR} GTEST_COPY_DEST)
        endif()
        add_custom_command(TARGET vk_loader_validation_tests POST_BUILD
            COMMAND xcopy /Y /I ${GTEST_COPY_SRC1} ${GTEST_COPY_DEST}
            COMMAND xcopy /Y /I ${GTEST_COPY_SRC2} ${GTEST_COPY_DEST}
        )
        # Copy the loader shared lib (if built) to the test application directory so the test app finds it.
        if((NOT ENABLE_STATIC_LOADER) AND TARGET vulkan)
            add_custom_command(TARGET vk_loader_validation_tests POST_BUILD
                COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:vulkan> $<TARGET_FILE_DIR:vk_loader_validation_tests>)
        endif()
    endif()

    add_subdirectory(layers)
endif()
