# Target UWP (Xbox Series X, dev mode) de DKC1Recomp: presentador Direct3D 11.
# Reusa el motor + el recomp generado + el runner DKC1 del clon del fuente, y
# cambia la capa de host por uwp_host.cpp. NO usa SDL ni OpenGL.
#
# Configurar (fuera del árbol, ruta corta por MAX_PATH), con el toolset UWP:
#   cmake -S <este dir> -B <build> -G Ninja ^
#     -DCMAKE_SYSTEM_NAME=WindowsStore -DCMAKE_SYSTEM_VERSION=10.0 ^
#     -DCMAKE_BUILD_TYPE=Release -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded ^
#     -DDKC1_SRC=<ruta a tu copia del proyecto>
#   cmake --build <build> --target dkc1_uwp
cmake_minimum_required(VERSION 3.20)
project(dkc1_uwp C CXX)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(NOT DEFINED DKC1_SRC)
    message(FATAL_ERROR "Pasa -DDKC1_SRC=<ruta a tu copia del proyecto>")
endif()

set(SNESRECOMP_ROOT ${DKC1_SRC}/snesrecomp)
# runner.cmake valida el valor aunque no enlacemos SDL (no llamamos snesrecomp_target_sdl).
set(SNESRECOMP_SDL_BACKEND "SDL2" CACHE STRING "" FORCE)
include(${SNESRECOMP_ROOT}/runner/runner.cmake)

file(GLOB DKC1_GEN CONFIGURE_DEPENDS "${DKC1_SRC}/generated/snesrecomp/*.c")
if(NOT DKC1_GEN)
    message(FATAL_ERROR "No hay fuentes generadas en ${DKC1_SRC}/generated/snesrecomp")
endif()

# Igual que el proyecto: DKC1 trae sus propios keybinds/host_report (headless_host.c).
set(DKC1_ENGINE ${SNESRECOMP_RUNNER_SOURCES})
list(REMOVE_ITEM DKC1_ENGINE
    "${SNESRECOMP_ROOT}/runner/src/keybinds.c"
    "${SNESRECOMP_ROOT}/runner/src/host_report.c")

set(DKC1_RUNNER
    ${DKC1_SRC}/runner/dkc1_game.c
    ${DKC1_SRC}/runner/dkc1_video.c
    ${DKC1_SRC}/runner/dkc1_baby_kong.c
    ${DKC1_SRC}/runner/dkc1_baby_kong_animation.c
    ${DKC1_SRC}/runner/dkc1_baby_kong_movement.c
    ${DKC1_SRC}/runner/dkc1_margin_proxy.c
    ${DKC1_SRC}/runner/dkc1_ws_trace.c
    ${DKC1_SRC}/runner/input_playback.c
    ${DKC1_SRC}/runner/wram_dump.c
    ${DKC1_SRC}/runner/dkc1_script.c
    ${DKC1_SRC}/runner/dkc1_debug_dump.c
    ${DKC1_SRC}/runner/dkc1_flight_recorder.c
    ${DKC1_SRC}/runner/dkc1_blank_scan.c
    ${DKC1_SRC}/runner/dkc1_invariant_monitor.c
    ${DKC1_SRC}/runner/verified_rom.c)

add_executable(dkc1_uwp WIN32
    ${DKC1_ENGINE}
    ${DKC1_GEN}
    ${DKC1_RUNNER}
    ${DKC1_SRC}/runner/headless_host.c
    ${CMAKE_CURRENT_SOURCE_DIR}/uwp_host.cpp)

target_include_directories(dkc1_uwp PRIVATE
    ${DKC1_SRC}/runner
    ${DKC1_SRC}/recomp
    ${SNESRECOMP_RUNNER_INCLUDE_DIRS}
    ${CMAKE_CURRENT_SOURCE_DIR})

target_compile_definitions(dkc1_uwp PRIVATE
    SNESRECOMP_TRACE=0
    SNESRECOMP_REVERSE_DEBUG=0
    SNESRECOMP_EXTERNAL_RAM_ROUTINE_GUARDS=1
    SYSTEM_VOLUME_MIXER_AVAILABLE=0
    _CRT_SECURE_NO_WARNINGS
    DKC1_BUILD_COMMIT="uwp"
    DKC1_BUILD_CONFIG="uwp")

target_compile_options(dkc1_uwp PRIVATE
    $<$<COMPILE_LANGUAGE:C>:/W0>
    $<$<COMPILE_LANGUAGE:CXX>:/EHsc>
    $<$<COMPILE_LANGUAGE:CXX>:/bigobj>)

# Enlace estilo receta probada de Mario Kart Wii (prueba-uwp): CRT «store» (lo trae el
# entorno vcvarsall x64 store + /MD) + OneCore, sin user32 (MessageBoxA se quitó de
# rom_image_verify.c). d3d11/dxgi/xaudio2 son válidos en AppContainer.
target_link_libraries(dkc1_uwp PRIVATE
    ${SNESRECOMP_RUNNER_LIBRARIES}
    d3d11 dxgi xaudio2 WindowsApp OneCore)

# El generador Ninja de WindowsStore NO añade /APPCONTAINER solo; sin ese bit el
# loader de UWP rechaza el paquete al activarlo. Se fuerza aquí.
target_link_options(dkc1_uwp PRIVATE /APPCONTAINER)

set_target_properties(dkc1_uwp PROPERTIES OUTPUT_NAME DKC1Recomp)
