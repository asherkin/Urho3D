#
# Copyright (c) 2008-2013 the Urho3D project.
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
# THE SOFTWARE.
#

if (CMAKE_PROJECT_NAME STREQUAL ExternalProject-tolua++)
    # Do not install the buildvm host tool from tolua++ external project because it is built for targetting host system itself
    # We only want to keep the buildvm host tool which is built for cross-compiling target (the one built from internal LuaJIT CMake-target)
    set (DEST_RUNTIME_DIR "")   # In this particular case, it is not equivalent to unset() the variable
elseif (NOT CMAKE_PROJECT_NAME STREQUAL Urho3D)
    # Set project name
    project (ExternalProject-buildvm)

    # Set minimum version
    cmake_minimum_required (VERSION 2.8.6)

    if (COMMAND cmake_policy)
        cmake_policy (SET CMP0003 NEW)
        if (CMAKE_VERSION VERSION_GREATER 2.8.12 OR CMAKE_VERSION VERSION_EQUAL 2.8.12)
            # INTERFACE_LINK_LIBRARIES defines the link interface
            cmake_policy (SET CMP0022 NEW)
        endif ()
        if (CMAKE_VERSION VERSION_GREATER 3.0.0 OR CMAKE_VERSION VERSION_EQUAL 3.0.0)
            # Disallow use of the LOCATION target property - therefore we set to OLD as we still need it
            cmake_policy (SET CMP0026 OLD)
            # MACOSX_RPATH is enabled by default
            cmake_policy (SET CMP0042 NEW)
        endif ()
    endif ()

    # Set CMake modules search path
    set (CMAKE_MODULE_PATH ${BAKED_CMAKE_SOURCE_DIR}/CMake/Modules)

    # Include Urho3D Cmake common module
    include (Urho3D-CMake-common)

    # Unescape the variables
    foreach (ESCAPED_VAR HOST_XCFLAGS TARGET_ARCH DASM_FLAGS DASM_ARCH)
        string (REPLACE +D -D ${ESCAPED_VAR} ${${ESCAPED_VAR}})
        string (REPLACE , ";" ${ESCAPED_VAR} ${${ESCAPED_VAR}})
    endforeach ()
endif ()

# Add dependency target
add_subdirectory (minilua)

# Define target name
set (TARGET_NAME buildvm)

# Add definitions specific for host C compiler
add_definitions (${HOST_XCFLAGS} ${TARGET_ARCH})

# Define generated source files
set (GEN_ARCH_H ${CMAKE_CURRENT_BINARY_DIR}/buildvm_arch.h)
set (DASM_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../../dynasm)
set (DASM ${DASM_DIR}/dynasm.lua)
set (DASM_DASC ${CMAKE_CURRENT_SOURCE_DIR}/../vm_${DASM_ARCH}.dasc)
file (GLOB DASM_LUA ${DASM_DIR}/*.lua)
add_custom_command (OUTPUT ${GEN_ARCH_H}
    COMMAND ${CMAKE_BINARY_DIR}/bin/tool/minilua ${DASM} ${DASM_FLAGS} -o ${GEN_ARCH_H} ${DASM_DASC}
    DEPENDS minilua ${DASM_LUA} ${DASM_DASC}
    WORKING_DIRECTORY ${DASM_DIR}
    COMMENT "Generating buildvm arch header file")

# Define source files
file (GLOB C_FILES buildvm*.c)
set (SOURCE_FILES ${C_FILES} ${GEN_ARCH_H})

# Define dependency libs
set (INCLUDE_DIRS ${CMAKE_CURRENT_BINARY_DIR} ..)

# All tools must be natively built and output to bin/tool subdir to differentiate them from platform-specific binaries in the bin directory
set_output_directories (${CMAKE_BINARY_DIR}/bin/tool RUNTIME PDB)
if (DEST_RUNTIME_DIR STREQUAL bin)
    set (DEST_RUNTIME_DIR bin/tool)
endif ()

# Setup target
setup_executable (NODEPS)
