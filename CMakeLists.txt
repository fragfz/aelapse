cmake_minimum_required(VERSION 3.15)

set(CMAKE_OSX_ARCHITECTURES "arm64;x86_64" CACHE STRING "" FORCE)
set(CMAKE_OSX_DEPLOYMENT_TARGET "10.10" CACHE STRING "" FORCE)
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

project(Aelapse VERSION 1.0.0)
set(COMPANY_NAME "S-audio")

option(MOD_DWARF "Build headless (no GUI/X11) for MOD Dwarf and similar embedded targets" ON)

set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_RELEASE TRUE)

find_package(JUCE CONFIG QUIET)
if(NOT ${JUCE_FOUND})
    Include(FetchContent)

    FetchContent_Declare(
      JUCE
      GIT_REPOSITORY https://github.com/juce-framework/JUCE
      GIT_TAG        7.0.12
    )
    FetchContent_MakeAvailable(JUCE)
endif()

juce_add_plugin(${PROJECT_NAME}
    COMPANY_NAME "${COMPANY_NAME}"
    IS_SYNTH FALSE
    NEEDS_MIDI_INPUT FALSE
    NEEDS_MIDI_OUTPUT FALSE
    IS_MIDI_EFFECT FALSE
    EDITOR_WANTS_KEYBOARD_FOCUS FALSE
    # COPY_PLUGIN_AFTER_BUILD TRUE/FALSE
    PLUGIN_MANUFACTURER_CODE Saud
    PLUGIN_CODE Aeth
    LV2URI https://github.com/smiarx/aelapse

    FORMATS LV2
    PRODUCT_NAME "${PROJECT_NAME}")

# tests
include(CTest)
option(BUILD_TESTING "Build tests" ON)
if (BUILD_TESTING)
   enable_testing()
endif()

# set dsp options itnernally
set(DSP_SPRINGS_RMS ON CACHE INTERNAL "")
set(DSP_SPRINGS_SHAKE ON CACHE INTERNAL "")
set(DSP_TAPEDELAY_SWITCH_INDICATOR ON CACHE INTERNAL "")

add_subdirectory(submodules/dsp/)
add_subdirectory(submodules/readerwriterqueue/)
add_subdirectory(src/)

target_compile_definitions(${PROJECT_NAME}
    PUBLIC
        JUCE_WEB_BROWSER=0  # If you remove this, add `NEEDS_WEB_BROWSER TRUE` to the `juce_add_plugin` call
        JUCE_USE_CURL=0     # If you remove this, add `NEEDS_CURL TRUE` to the `juce_add_plugin` call
        JUCE_VST3_CAN_REPLACE_VST2=0
        JUCE_DISPLAY_SPLASH_SCREEN=0
        PROJECT_NAME="${PROJECT_NAME}"
        COMPANY_NAME="${COMPANY_NAME}"
        SPRINGS_RMS
        SPRINGS_SHAKE
        TAPEDELAY_SWITCH_INDICATOR
        $<$<BOOL:${MOD_DWARF}>:MOD_DWARF>
        JUCE_AUDIOPROCESSOR_NO_GUI=1
    )

target_compile_features(${PROJECT_NAME} PUBLIC cxx_std_17)

#ignore omp simd pragma
if(MSVC)
    target_compile_options(${PROJECT_NAME} PRIVATE
        /openmp
    )
else()
    target_compile_options(${PROJECT_NAME} PRIVATE
        -fopenmp-simd
    )
endif()

set_property(TARGET readerwriterqueue PROPERTY SYSTEM TRUE)
set_property(TARGET dsp_include PROPERTY SYSTEM TRUE)

target_link_libraries(${PROJECT_NAME}
    PRIVATE
        ${PROJECT_NAME}_factory
        juce::juce_audio_utils
        juce::juce_audio_plugin_client
        readerwriterqueue
        dsp_springs_processor
        dsp_tapedelay_processor
    PUBLIC
        juce::juce_recommended_config_flags
        juce::juce_recommended_warning_flags
    )

if(NOT MOD_DWARF)
    target_link_libraries(${PROJECT_NAME}
        PRIVATE
            ${PROJECT_NAME}_assets
            ${PROJECT_NAME}_fonts
            juce::juce_opengl
        )
endif()
