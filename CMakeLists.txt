# Copyright 2012-2019 Ronald Römer
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

cmake_minimum_required(VERSION 3.1)
project(vtkbool VERSION 2.7.2)

if(DEFINED VTK_MODULE_ENABLE_VTK_Boolean)
message(STATUS "Boolean: Building as a Remote VTK module")
set(Module_SRCS
    vtkPolyDataBooleanFilter.cxx
    vtkPolyDataContactFilter.cxx
    Utilities.cxx
    libs/decomp/Decomposer.cxx
    libs/merger/Merger.cxx
    libs/vp/RmTrivials.cxx
    libs/vp/VisPoly.cxx
    libs/vp/Tools.cxx  
    )
set(VTK_BUILD_TESTING "YES")
include_directories(libs/merger libs/decomp libs/vp libs/aabb)
if (VTK_VERSION VERSION_LESS "8.90.0")
  vtk_module_library(Boolean ${Module_SRCS})
else()
  vtk_module_add_module(VTK::Boolean
    SOURCES ${Module_SRCS}
    )
endif()
set_source_files_properties(
    Merger.cxx Utilities.cxx
    PROPERTIES WRAP_EXCLUDE 1
    )
if(MSVC)
  target_compile_options(Boolean PUBLIC /DWIN32 /D_WINDOWS /W3 /GR /EHsc)
#    add_compile_options(/EHsc)
#    add_definitions(-D_SCL_SECURE_NO_WARNINGS)
#    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
else()
    target_compile_options(Boolean PUBLIC -Wall -Wextra -Wshadow -Woverloaded-virtual -Wno-deprecated -Wno-strict-overflow -Wno-array-bounds)
endif()
include_directories(libs/aabb)
else()
message(STATUS "vtkbool: Building as an independent library")
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_CXX_OUTPUT_EXTENSION_REPLACE 1)

if(MSVC)
    add_compile_options(/EHsc)
    add_definitions(-D_SCL_SECURE_NO_WARNINGS)
    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
else()
    add_compile_options(-Wno-deprecated -fPIC -Wunused)
endif()

option(VTKBOOL_PARAVIEW "" OFF)
option(VTKBOOL_DEBUG "" OFF)
option(VTKBOOL_TESTING "" OFF)
option(VTKBOOL_COVERAGE "" OFF)

mark_as_advanced(VTKBOOL_DEBUG)
mark_as_advanced(VTKBOOL_TESTING)
mark_as_advanced(VTKBOOL_COVERAGE)

if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU" AND VTKBOOL_COVERAGE)
    add_compile_options(-fprofile-arcs -ftest-coverage)
    set(CMAKE_EXE_LINKER_FLAGS "-fprofile-arcs")
    set(CMAKE_SHARED_LINKER_FLAGS "-fprofile-arcs")
endif()

set(srcs
    vtkPolyDataBooleanFilter.cxx
    vtkPolyDataContactFilter.cxx
    )

include_directories(".")

if(VTKBOOL_DEBUG)
    add_definitions(-DDEBUG)
endif()

if(VTKBOOL_PARAVIEW)
    set(VTKBOOL_TESTING OFF CACHE BOOL "" FORCE)

    add_subdirectory(paraview_plugin ${CMAKE_BINARY_DIR}/paraview_plugin)

else()

    #unset(VTK_DIR CACHE)

    find_package(VTK REQUIRED COMPONENTS vtkFiltersSources vtkIOLegacy vtkFiltersExtraction vtkFiltersGeometry vtkFiltersModeling vtkRenderingFreeType OPTIONAL_COMPONENTS vtkWrappingPythonCore NO_MODULE)

    if(VTK_FOUND)
        include(${VTK_USE_FILE})

        add_subdirectory(libs)

        include_directories(libs/merger libs/decomp libs/vp libs/aabb)

        add_library(${PROJECT_NAME} SHARED ${srcs})
        target_link_libraries(${PROJECT_NAME} ${VTK_LIBRARIES} decomp merger)

        if(VTK_INSTALL_PREFIX)
            set(_vtk ${VTK_INSTALL_PREFIX})
        else()
            set(_vtk ${VTK_DIR})
        endif()

        if(WIN32)
            string(REPLACE "/" "\\" _vtk_win ${_vtk})
        endif()

        if(VTKBOOL_TESTING)
            enable_testing()

            add_executable(testing testing/testing.cxx)
            target_link_libraries(testing ${PROJECT_NAME} ${VTK_LIBRARIES})

            foreach(i RANGE 19)
                add_test(NAME Test_${i} COMMAND testing ${i})

                if(WIN32)
                    set_property(TEST Test_${i} PROPERTY ENVIRONMENT "PATH=${_vtk_win}\\bin")
                endif()

            endforeach()

        endif()

        if(VTK_WRAP_PYTHON)

            if(${VTK_MAJOR_VERSION} GREATER 7)
                set(MODULE_HIERARCHY_NAME ${PROJECT_NAME}Hierarchy)
                set(${PROJECT_NAME}_WRAP_DEPENDS vtkCommonExecutionModel)
                include(vtkWrapHierarchy)
                vtk_wrap_hierarchy(${PROJECT_NAME} ${CMAKE_CURRENT_BINARY_DIR} vtkPolyDataBooleanFilter.h)
                set(KIT_HIERARCHY_FILE ${CMAKE_CURRENT_BINARY_DIR}/${MODULE_HIERARCHY_NAME}.txt)
                set(LIB_HIERARCHY_STAMP ${CMAKE_CURRENT_BINARY_DIR}/${MODULE_HIERARCHY_NAME}.stamp.txt)
            endif()

            set(ENV{LD_LIBRARY_PATH} "${_vtk}/lib")

            execute_process(COMMAND "${_vtk}/bin/vtkpython" -c "import sys; sys.stdout.write('.'.join(map(str, sys.version_info[:2])))" OUTPUT_VARIABLE py_ver)

            if(py_ver)
                if(CMAKE_VERSION VERSION_LESS "3.12")
                    find_package(PythonInterp ${py_ver} EXACT)
                    find_package(PythonLibs ${py_ver} EXACT)

                    if(PYTHONLIBS_FOUND)

                        include_directories(${PYTHON_INCLUDE_DIRS} ${PROJECT_SOURCE_DIR})

                        include(vtkWrapPython)

                        vtk_wrap_python3(${PROJECT_NAME}Python ${PROJECT_NAME}Python_srcs vtkPolyDataBooleanFilter.cxx)

                        add_library(${PROJECT_NAME}PythonD ${${PROJECT_NAME}Python_srcs})
                        target_link_libraries(${PROJECT_NAME}PythonD ${PROJECT_NAME} vtkWrappingPythonCore vtkCommonExecutionModelPythonD)

                        python_add_module(${PROJECT_NAME}Python PyInit.cxx ${PROJECT_NAME}PythonInit.cxx)
                        target_link_libraries(${PROJECT_NAME}Python PRIVATE ${PROJECT_NAME}PythonD)

                        if(VTKBOOL_TESTING)
                            add_test(NAME Test_Py COMMAND ${PYTHON_EXECUTABLE}
                                ${CMAKE_SOURCE_DIR}/testing/test_py_module.py
                                ${_vtk}/lib/python${py_ver}/site-packages
                                ${CMAKE_BINARY_DIR}
                                $<CONFIG>)

                            if(WIN32)
                                set_property(TEST Test_Py PROPERTY ENVIRONMENT "PATH=${_vtk_win}\\bin")
                            else()
                                set_property(TEST Test_Py PROPERTY ENVIRONMENT "LD_LIBRARY_PATH=${_vtk}/lib")
                            endif()
                        endif()

                    endif()

                else()
                    find_package(Python ${py_ver} EXACT COMPONENTS Interpreter Development)

                    if(Python_Development_FOUND)

                        include_directories(${PYTHON_INCLUDE_DIRS} ${PROJECT_SOURCE_DIR})

                        include(vtkWrapPython)

                        vtk_wrap_python3(${PROJECT_NAME}Python ${PROJECT_NAME}Python_srcs vtkPolyDataBooleanFilter.cxx)

                        add_library(${PROJECT_NAME}PythonD ${${PROJECT_NAME}Python_srcs})
                        target_link_libraries(${PROJECT_NAME}PythonD ${PROJECT_NAME} vtkWrappingPythonCore vtkCommonExecutionModelPythonD)

                        Python_add_library(${PROJECT_NAME}Python MODULE PyInit.cxx ${PROJECT_NAME}PythonInit.cxx)
                        target_link_libraries(${PROJECT_NAME}Python PRIVATE ${PROJECT_NAME}PythonD)

                        if(VTKBOOL_TESTING)
                            add_test(NAME Test_Py COMMAND ${Python_EXECUTABLE}
                                ${CMAKE_SOURCE_DIR}/testing/test_py_module.py
                                ${VTK_PYTHONPATH}
                                ${CMAKE_BINARY_DIR}
                                $<CONFIG>)

                            if(WIN32)
                                set_property(TEST Test_Py PROPERTY ENVIRONMENT "PATH=${_vtk_win}\\bin")
                            else()
                                set_property(TEST Test_Py PROPERTY ENVIRONMENT "LD_LIBRARY_PATH=${_vtk}/lib")
                            endif()
                        endif()
                    endif()

                endif()

            endif()

        endif()

    endif(VTK_FOUND)

endif()
endif()
