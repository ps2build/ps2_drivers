cmake_minimum_required(VERSION 3.13)

# Set the toolchain file before project()
if(NOT CMAKE_TOOLCHAIN_FILE)
    if(DEFINED ENV{PS2DEV})
        set(CMAKE_TOOLCHAIN_FILE "$ENV{PS2DEV}/cmake/ps2dev.cmake" CACHE FILEPATH "Toolchain file")
    else()
        message(FATAL_ERROR "PS2DEV environment variable is not set. Please set it to your PS2DEV installation path.")
    endif()
endif()

project(ps2_drivers VERSION 1.0.0 LANGUAGES C)

# Options
option(BUILD_SAMPLES "Build sample projects" ON)

# ============================================================================
# Real per-package resolution
# ============================================================================
# This project used to depend on a flat $PS2SDK tree (${PS2SDK}/iop/irx/
# for pre-built IRX blobs, ${PS2SDK}/ee/lib/ for EE-side archives) that
# no longer exists at all in the per-package ps2dev-suite distribution -
# ps2dev.cmake deliberately stopped defining a PS2SDK variable once
# ps2gl/ps2stuff moved off it ("There is no more flat $PS2SDK tree to
# point at" - its own comment). Every real dependency now lives at
# $PS2DEV/packages/<tier>/<name>/{bin,lib}/ instead - resolve from
# there directly.
set(PS2_PACKAGES "$ENV{PS2DEV}/packages")

# ps2_pkg_irx(<out_var> <tier> <pkg_name> <irx_filename>)
# Sets <out_var> to the real installed path if it exists, or to
# "<out_var>-NOTFOUND" (and a real STATUS message) if the package
# hasn't been migrated/built yet - callers gate on this instead of
# assuming every driver is always available.
function(ps2_pkg_irx out_var tier pkg_name irx_filename)
    set(_path "${PS2_PACKAGES}/${tier}/${pkg_name}/bin/${irx_filename}")
    if(EXISTS "${_path}")
        set(${out_var} "${_path}" PARENT_SCOPE)
    else()
        set(${out_var} "${out_var}-NOTFOUND" PARENT_SCOPE)
        message(STATUS "ps2_drivers: ${tier}/${pkg_name} (${irx_filename}) not installed - skipping the driver(s) that need it")
    endif()
endfunction()

# ps2_pkg_lib(<out_var> <tier> <pkg_name> <lib_filename>)
# Same as ps2_pkg_irx, for EE-side .a archives.
function(ps2_pkg_lib out_var tier pkg_name lib_filename)
    set(_path "${PS2_PACKAGES}/${tier}/${pkg_name}/lib/${lib_filename}")
    if(EXISTS "${_path}")
        set(${out_var} "${_path}" PARENT_SCOPE)
    else()
        set(${out_var} "${out_var}-NOTFOUND" PARENT_SCOPE)
        message(STATUS "ps2_drivers: ${tier}/${pkg_name} (${lib_filename}) not installed - final library may fail to link")
    endif()
endfunction()

# Set the bin2c tool path - a real ps2build-managed component now
# (ps2build update bin2c), not part of any package tree.
if(NOT DEFINED BIN2C)
    set(BIN2C "$ENV{PS2DEV}/bin/tools/bin2c.exe")
endif()
if(NOT EXISTS "${BIN2C}")
    message(FATAL_ERROR "bin2c not found at ${BIN2C} - run 'ps2build update bin2c'.")
endif()

set(INSTALL_LIB_DIR "${PS2_PACKAGES}/world/ps2_drivers/lib" CACHE PATH "Installation directory for libraries")
set(INSTALL_INC_DIR "${PS2_PACKAGES}/world/ps2_drivers/include" CACHE PATH "Installation directory for headers")
set(INSTALL_PKGCONFIG_DIR "${PS2_PACKAGES}/world/ps2_drivers/lib/pkgconfig" CACHE PATH "Installation directory for pkgconfig (.pc) files")

# Prepare git metadata
set(CUR_GIT_TAG v0.0.0)
set(CUR_GIT_COMMIT_HASH "")
set(CUR_GIT_COMMIT_TIMESTAMP "")
set(CUR_GIT_REFS "")
set(CUR_GIT_REMOTE_URL "")
find_package(Git)
if(GIT_FOUND)
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/.git)
        execute_process(COMMAND ${GIT_EXECUTABLE} describe --abbrev=0 --tags
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE TMP_GIT_TAG
        RESULT_VARIABLE TMP_RES
        OUTPUT_STRIP_TRAILING_WHITESPACE)
        execute_process(COMMAND ${GIT_EXECUTABLE} show --no-patch --format=%H
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE TMP_GIT_COMMIT_HASH
        RESULT_VARIABLE TMP_RES
        OUTPUT_STRIP_TRAILING_WHITESPACE)
        if(TMP_RES EQUAL "0")
            set(CUR_GIT_COMMIT_HASH ${TMP_GIT_COMMIT_HASH})
        endif()
        execute_process(COMMAND ${GIT_EXECUTABLE} show --no-patch --format=%cd --date=unix
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE TMP_GIT_COMMIT_TIMESTAMP
        RESULT_VARIABLE TMP_RES
        OUTPUT_STRIP_TRAILING_WHITESPACE)
        if(TMP_RES EQUAL "0")
            set(CUR_GIT_COMMIT_TIMESTAMP ${TMP_GIT_COMMIT_TIMESTAMP})
        endif()
        execute_process(COMMAND ${GIT_EXECUTABLE} show --no-patch "--format=%(decorate:prefix=,suffix=,tag=,separator=%x2C,pointer=%x2C)"
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE TMP_GIT_REFS
        RESULT_VARIABLE TMP_RES
        OUTPUT_STRIP_TRAILING_WHITESPACE)
        if(TMP_RES EQUAL "0")
            set(CUR_GIT_REFS ${TMP_GIT_REFS})
        endif()
        execute_process(COMMAND ${GIT_EXECUTABLE} config --get remote.origin.url
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE TMP_GIT_REMOTE_URL
        RESULT_VARIABLE TMP_RES
        OUTPUT_STRIP_TRAILING_WHITESPACE)
        if(TMP_RES EQUAL "0")
            set(CUR_GIT_REMOTE_URL ${TMP_GIT_REMOTE_URL})
        endif()
    endif()
endif()

# Set output library names
set(EE_LIB_IMPL "libps2_drivers_impl.a")
set(EE_LIB_COMBINED "libps2_drivers.a")

# Compiler flags (matching the Makefile)
add_compile_options(-Werror)
include_directories(${CMAKE_SOURCE_DIR}/include)

# ============================================================================
# Function: add_irx_source
# Converts a real, installed IRX file to a C source file using bin2c -
# a no-op (out_var left NOTFOUND) if the package isn't installed yet.
# ============================================================================
function(add_irx_source tier pkg_name irx_filename output_var)
    ps2_pkg_irx(_irx_path ${tier} ${pkg_name} ${irx_filename})
    if(NOT _irx_path)
        set(${output_var} "" PARENT_SCOPE)
        return()
    endif()

    # IRX file names may contain hyphens; the resulting C symbol must
    # use underscores to be a valid identifier (matches EXTERN_IRX()'s
    # own real naming convention in the driver sources).
    get_filename_component(irx_stem "${irx_filename}" NAME_WE)
    string(REPLACE "-" "_" irx_symbol "${irx_stem}")
    set(output_c "${CMAKE_CURRENT_BINARY_DIR}/${irx_symbol}_irx.c")

    add_custom_command(
        OUTPUT ${output_c}
        COMMAND ${BIN2C} ${_irx_path} ${output_c} ${irx_symbol}_irx
        DEPENDS ${_irx_path}
        COMMENT "Converting ${irx_filename} to C source"
        VERBATIM
    )

    set(${output_var} ${output_c} PARENT_SCOPE)
endfunction()

# ============================================================================
# Function: add_driver_objects
# Compiles a driver source file multiple times with different defines
# ============================================================================
function(add_driver_objects driver_name)
    # Parse the function names from remaining arguments
    set(functions ${ARGN})
    set(driver_objs "")

    foreach(func IN LISTS functions)
        set(obj_name "${func}_ps2_${driver_name}_driver")
        add_library(${obj_name} OBJECT "${CMAKE_SOURCE_DIR}/src/ps2_${driver_name}_driver.c")
        target_compile_definitions(${obj_name} PRIVATE "F_${func}_ps2_${driver_name}_driver")
        target_include_directories(${obj_name} PRIVATE "${CMAKE_SOURCE_DIR}/include")
        list(APPEND driver_objs $<TARGET_OBJECTS:${obj_name}>)
    endforeach()

    set(${driver_name}_DRIVER_OBJECTS ${driver_objs} PARENT_SCOPE)
endfunction()

# ============================================================================
# Resolve real IRX sources - each may be unavailable (NOTFOUND) if its
# package hasn't been migrated/built yet. Real tier/name/filename per
# package.yaml, not guessed - see each artifact's own irx: field.
# ============================================================================
add_irx_source(core sio2man sio2man.irx SIO2MAN_IRX)
add_irx_source(core iomanx iomanX.irx IOMANX_IRX)
add_irx_source(core filexio fileXio.irx FILEXIO_IRX)
add_irx_source(core mcman mcman.irx MCMAN_IRX)
add_irx_source(core mcserv mcserv.irx MCSERV_IRX)
add_irx_source(core bdm bdm.irx BDM_IRX)
add_irx_source(core bdmfs_fatfs bdmfs_fatfs.irx BDMFS_FATFS_IRX)
add_irx_source(core usbd usbd.irx USBD_IRX)
add_irx_source(core usbmass_bd usbmass_bd.irx USBMASS_BD_IRX)   # not yet migrated
add_irx_source(core mx4sio_bd mx4sio_bd.irx MX4SIO_BD_IRX)      # not yet migrated
add_irx_source(core cdfs cdfs.irx CDFS_IRX)                     # not yet migrated
add_irx_source(core dev9 ps2dev9.irx PS2DEV9_IRX)                # not yet migrated
add_irx_source(core atad ps2atad.irx PS2ATAD_IRX)                 # not yet migrated
add_irx_source(core hdd ps2hdd.irx PS2HDD_IRX)                   # not yet migrated
add_irx_source(core fs ps2fs.irx PS2FS_IRX)                    # not yet migrated
add_irx_source(core mtapman mtapman.irx MTAPMAN_IRX)
add_irx_source(core padman padman.irx PADMAN_IRX)
add_irx_source(core libsd libsd.irx LIBSD_IRX)
add_irx_source(world audsrv audsrv.irx AUDSRV_IRX)
add_irx_source(core poweroff poweroff.irx POWEROFF_IRX)
add_irx_source(core mouse ps2mouse.irx PS2MOUSE_IRX)
add_irx_source(core kbd ps2kbd.irx PS2KBD_IRX)
add_irx_source(core ps2cam ps2cam.irx PS2CAM_IRX)
add_irx_source(core netman netman.irx NETMAN_IRX)
add_irx_source(core smap smap.irx SMAP_IRX)
add_irx_source(core ps2ip-nm ps2ip-nm.irx PS2IP_NM_IRX)          # deferred: real IOP-side lwip port
add_irx_source(core ps2ips-iop ps2ips.irx PS2IPS_IRX)            # deferred: real IOP-side lwip port

set(IRX_SOURCES "")
foreach(_v SIO2MAN_IRX IOMANX_IRX FILEXIO_IRX MCMAN_IRX MCSERV_IRX BDM_IRX
          BDMFS_FATFS_IRX USBD_IRX USBMASS_BD_IRX MX4SIO_BD_IRX CDFS_IRX
          PS2DEV9_IRX PS2ATAD_IRX PS2HDD_IRX PS2FS_IRX MTAPMAN_IRX
          PADMAN_IRX LIBSD_IRX AUDSRV_IRX POWEROFF_IRX PS2MOUSE_IRX
          PS2KBD_IRX PS2CAM_IRX NETMAN_IRX SMAP_IRX PS2IP_NM_IRX PS2IPS_IRX)
    if(${_v})
        list(APPEND IRX_SOURCES ${${_v}})
    endif()
endforeach()

# ============================================================================
# Generate driver objects - each gated on the real IRX(es) it embeds
# actually being available (a driver .c file calls EXTERN_IRX() on
# each one by name, so it can't even compile, let alone work, without
# the matching embed above).
# ============================================================================
set(_ps2_drivers_available "")
set(_ps2_drivers_skipped "")
set(ALL_DRIVER_OBJECTS "")

macro(_maybe_add_driver name)
    if(${ARGN})
        add_driver_objects(${name} ${_driver_funcs_${name}})
        list(APPEND ALL_DRIVER_OBJECTS ${${name}_DRIVER_OBJECTS})
        list(APPEND _ps2_drivers_available ${name})
    else()
        list(APPEND _ps2_drivers_skipped ${name})
    endif()
endmacro()

set(_driver_funcs_sio2man internals init deinit)
set(_driver_funcs_fileXio internals init deinit)
set(_driver_funcs_memcard internals init deinit)
set(_driver_funcs_bdm internals init deinit)
set(_driver_funcs_usbd internals init deinit)
set(_driver_funcs_usb internals init deinit)
set(_driver_funcs_mx4sio internals init deinit)
set(_driver_funcs_cdfs internals init deinit)
set(_driver_funcs_dev9 internals init deinit)
set(_driver_funcs_hdd internals init deinit mount_hdd_partition mount_current_partition umount_hdd_partition umount_current_partition)
set(_driver_funcs_filesystem waitUntilDeviceIsReady rootDevicePath getBootDeviceID __internal_deinit deinit init __internal_deinit_only_boot deinit_only_boot init_only_boot)
set(_driver_funcs_joystick internals init deinit)
set(_driver_funcs_audio internals init deinit)
set(_driver_funcs_poweroff internals init deinit)
set(_driver_funcs_mouse internals init deinit)
set(_driver_funcs_keyboard internals init deinit)
set(_driver_funcs_camera internals init deinit)
set(_driver_funcs_netman internals init deinit)
set(_driver_funcs_smap internals init deinit)
set(_driver_funcs_eeip internals init deinit configure_eeip_network eeip_network_config_default_dhcp eeip_get_current_config)
set(_driver_funcs_iopip internals init deinit configure_iopip_network iopip_network_config_default iopip_get_current_config)

_maybe_add_driver(sio2man SIO2MAN_IRX)
_maybe_add_driver(fileXio IOMANX_IRX AND FILEXIO_IRX)
_maybe_add_driver(memcard MCMAN_IRX AND MCSERV_IRX)
_maybe_add_driver(bdm BDM_IRX AND BDMFS_FATFS_IRX)
_maybe_add_driver(usbd USBD_IRX)
_maybe_add_driver(usb BDMFS_FATFS_IRX AND USBMASS_BD_IRX)
_maybe_add_driver(mx4sio MX4SIO_BD_IRX)
_maybe_add_driver(cdfs CDFS_IRX)
_maybe_add_driver(dev9 PS2DEV9_IRX)
_maybe_add_driver(hdd PS2ATAD_IRX AND PS2HDD_IRX AND PS2FS_IRX)
_maybe_add_driver(filesystem TRUE)   # pure orchestration, no IRX of its own
_maybe_add_driver(joystick MTAPMAN_IRX AND PADMAN_IRX)
_maybe_add_driver(audio LIBSD_IRX AND AUDSRV_IRX)
_maybe_add_driver(poweroff POWEROFF_IRX)
_maybe_add_driver(mouse PS2MOUSE_IRX)
_maybe_add_driver(keyboard PS2KBD_IRX)
_maybe_add_driver(camera PS2CAM_IRX)
_maybe_add_driver(netman NETMAN_IRX)
_maybe_add_driver(smap SMAP_IRX)
_maybe_add_driver(eeip TRUE)   # EE-side lwip (core/ps2ip) needs no IRX of its own to load
_maybe_add_driver(iopip PS2IPS_IRX AND PS2IP_NM_IRX)

# ============================================================================
# Build the implementation library
# ============================================================================
add_library(ps2_drivers_impl STATIC
    ${IRX_SOURCES}
    ${ALL_DRIVER_OBJECTS}
)

set_target_properties(ps2_drivers_impl PROPERTIES
    OUTPUT_NAME "ps2_drivers_impl"
    ARCHIVE_OUTPUT_NAME "ps2_drivers_impl"
)

# bin2c's own generated *_irx.c files start with "#pragma once" (a
# normal header-guard idiom) but get compiled directly as their own
# translation unit here, not #include'd - triggers a real, pedantic
# -Werror=pragma-once-outside-header under this project's global
# -Werror. Real, harmless (the pragma has no effect either way for a
# file that's never included), not worth restructuring bin2c's own
# output for - just don't error on it for these specific generated
# files.
if(IRX_SOURCES)
    set_source_files_properties(${IRX_SOURCES} PROPERTIES
        COMPILE_OPTIONS "-Wno-error=pragma-once-outside-header"
    )
endif()

# ============================================================================
# Combine with real, installed per-package EE-side libraries
# ============================================================================
set(PS2SDK_LIBS "")
foreach(_pair
    "core;filexio;libfilexio.a"
    "core;mtap;libmtap.a"
    "core;pad;libpad.a"
    "world;audsrv;libaudsrv.a"
    "core;poweroff;libpoweroff.a"
    "core;mouse;libmouse.a"
    "core;kbd;libkbd.a"
    "core;ps2cam;libps2cam.a"
    "core;netman;libnetman.a"
    "core;ps2ip;libps2ip.a"
    "core;ps2ips;libps2ips.a"
)
    list(GET _pair 0 _tier)
    list(GET _pair 1 _name)
    list(GET _pair 2 _file)
    ps2_pkg_lib(_lib_path ${_tier} ${_name} ${_file})
    if(_lib_path)
        list(APPEND PS2SDK_LIBS "${_lib_path}")
    endif()
endforeach()

# Create the combined library using ar with MRI script
# Generate the MRI script content with all libraries to combine
set(MRI_SCRIPT_CONTENT "create ${CMAKE_CURRENT_BINARY_DIR}/${EE_LIB_COMBINED}\n")
set(MRI_SCRIPT_CONTENT "${MRI_SCRIPT_CONTENT}addlib $<TARGET_FILE:ps2_drivers_impl>\n")
foreach(lib ${PS2SDK_LIBS})
    set(MRI_SCRIPT_CONTENT "${MRI_SCRIPT_CONTENT}addlib ${lib}\n")
endforeach()
set(MRI_SCRIPT_CONTENT "${MRI_SCRIPT_CONTENT}save\nend\n")

# Write MRI script at configure time (generator expressions evaluated at build time via file(GENERATE))
file(GENERATE OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/mri_script.txt CONTENT "${MRI_SCRIPT_CONTENT}")

add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${EE_LIB_COMBINED}
    COMMAND ${CMAKE_COMMAND} -E rm -f ${CMAKE_CURRENT_BINARY_DIR}/${EE_LIB_COMBINED}
    COMMAND ${CMAKE_AR} -M < ${CMAKE_CURRENT_BINARY_DIR}/mri_script.txt
    DEPENDS ps2_drivers_impl ${PS2SDK_LIBS} ${CMAKE_CURRENT_BINARY_DIR}/mri_script.txt
    COMMENT "Combining libraries into ${EE_LIB_COMBINED}"
)

add_custom_target(ps2_drivers ALL
    DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${EE_LIB_COMBINED}
)

# ============================================================================
# Prepare pkg-config metadata file
# ============================================================================
set(PCKEY_NAME ${PROJECT_NAME})
set(PCKEY_DESCRIPTION "Convenience library for ps2sdk I/O drivers")
set(PCKEY_VERSION ${PROJECT_VERSION})
set(PCKEY_REQUIRES "")
set(PCKEY_LIBS "-lps2_drivers")
set(PCKEY_CFLAGS "")
set(PS2_DRIVERS_PC ${CMAKE_CURRENT_BINARY_DIR}/ps2_drivers.pc)
configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/ps2_drivers.pc.cmakein
    ${PS2_DRIVERS_PC} @ONLY)

# ============================================================================
# Install targets
# ============================================================================
if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    install(
        FILES ${CMAKE_CURRENT_BINARY_DIR}/${EE_LIB_COMBINED}
        DESTINATION "${INSTALL_LIB_DIR}"
    )
endif()

if(NOT SKIP_INSTALL_HEADERS AND NOT SKIP_INSTALL_ALL)
    install(
        DIRECTORY ${CMAKE_SOURCE_DIR}/include/
        DESTINATION "${INSTALL_INC_DIR}"
        FILES_MATCHING PATTERN "*.h"
    )
endif()

if(NOT SKIP_INSTALL_FILES AND NOT SKIP_INSTALL_ALL )
    install(
        FILES ${PS2_DRIVERS_PC}
        DESTINATION "${INSTALL_PKGCONFIG_DIR}"
    )
endif()

# ============================================================================
# Include samples if enabled
# ============================================================================
if(BUILD_SAMPLES)
    add_subdirectory(samples)
endif()

# ============================================================================
# Print configuration summary
# ============================================================================
message(STATUS "")
message(STATUS "ps2_drivers configuration:")
message(STATUS "  Version: ${PROJECT_VERSION}")
message(STATUS "  Build samples: ${BUILD_SAMPLES}")
message(STATUS "  Output library: ${EE_LIB_COMBINED}")
message(STATUS "  Install prefix: ${PS2_PACKAGES}/world/ps2_drivers")
list(SORT _ps2_drivers_available)
list(SORT _ps2_drivers_skipped)
message(STATUS "  Drivers available: ${_ps2_drivers_available}")
if(_ps2_drivers_skipped)
    message(STATUS "  Drivers skipped (real, not-yet-migrated sdk-core/sdk-world prerequisite): ${_ps2_drivers_skipped}")
endif()
message(STATUS "")
