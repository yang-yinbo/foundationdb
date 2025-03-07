set(SRCS
  fdb/__init__.py
  fdb/directory_impl.py
  fdb/impl.py
  fdb/locality.py
  fdb/six.py
  fdb/subspace_impl.py
  fdb/tenant_management.py
  fdb/tuple.py
  README.rst
  MANIFEST.in)

if(APPLE)
  list(APPEND SRCS fdb/libfdb_c.dylib.pth)
else()
  list(APPEND SRCS fdb/libfdb_c.so.pth)
endif()

set(out_files "")
foreach(src ${SRCS})
  get_filename_component(dirname ${src} DIRECTORY)
  get_filename_component(extname ${src} EXT)
  if(NOT EXISTS ${dirname})
    file(MAKE_DIRECTORY ${PROJECT_BINARY_DIR}/bindings/python/${dirname})
  endif()
  set(from_path ${CMAKE_CURRENT_SOURCE_DIR}/${src})
  set(to_path ${CMAKE_CURRENT_BINARY_DIR}/${src})
  add_custom_command(OUTPUT ${PROJECT_BINARY_DIR}/bindings/python/${src}
    COMMAND ${CMAKE_COMMAND} -E copy ${from_path} ${to_path}
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/${src}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    COMMENT "copy ${src}")
  set(out_files "${out_files};${PROJECT_BINARY_DIR}/bindings/python/${src}")
endforeach()
add_custom_target(python_binding ALL DEPENDS ${out_files})

file(MAKE_DIRECTORY ${PROJECT_BINARY_DIR}/bindings/python/fdb)
set(options_file ${PROJECT_BINARY_DIR}/bindings/python/fdb/fdboptions.py)
vexillographer_compile(TARGET fdb_python_options LANG python OUT ${options_file}
  OUTPUT ${options_file})

add_dependencies(python_binding fdb_python_options)

set(out_files "${out_files};${options_file}")
# TODO[mpilman]: it is not clear whether we want to have rpms for python
#install(FILES ${out_files} DESTINATION ${FDB_PYTHON_INSTALL_DIR} COMPONENT python)

# Create sdist
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/setup.py.cmake ${CMAKE_CURRENT_BINARY_DIR}/setup.py)
configure_file(${CMAKE_SOURCE_DIR}/LICENSE ${CMAKE_CURRENT_BINARY_DIR}/LICENSE COPYONLY)
find_program(pycodestyle pycodestyle)
if (pycodestyle)
  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/check_py_code_style
    COMMAND ${pycodestyle} bindings/python --config=${CMAKE_CURRENT_SOURCE_DIR}/setup.cfg &&
            ${CMAKE_COMMAND} -E ${CMAKE_CURRENT_BINARY_DIR}/check_py_code_style
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    DEPENDS ${out_files}
    COMMENT "Check python code style")
  add_custom_target(fdb_python_check DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/check_py_code_style)
else()
  add_custom_target(fdb_python_check COMMAND ${CMAKE_COMMAND} -E echo "Skipped Python style check! Missing: pycodestyle")
endif()
if(NOT FDB_RELEASE)
  set(not_fdb_release_string "a1")
else()
  set(not_fdb_release_string "")
endif()
set(setup_file_name foundationdb-${FDB_VERSION}.tar.gz)
set(package_file ${CMAKE_BINARY_DIR}/packages/foundationdb-${FDB_VERSION}${not_fdb_release_string}.tar.gz)
add_custom_command(OUTPUT ${package_file}
  COMMAND $<TARGET_FILE:Python3::Interpreter> setup.py sdist --formats=gztar &&
          ${CMAKE_COMMAND} -E copy dist/${setup_file_name} ${package_file}
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
  COMMENT "Create Python sdist package")
add_custom_target(python_package DEPENDS ${package_file})
add_dependencies(python_package python_binding)
add_dependencies(packages python_package)

if (NOT WIN32 AND NOT OPEN_FOR_IDE)
  add_fdbclient_test(
    NAME single_process_fdbcli_tests
    COMMAND ${CMAKE_SOURCE_DIR}/bindings/python/tests/fdbcli_tests.py
            ${CMAKE_BINARY_DIR}
            @CLUSTER_FILE@
            )
  add_fdbclient_test(
    NAME multi_process_fdbcli_tests
    PROCESS_NUMBER 5
    COMMAND ${CMAKE_SOURCE_DIR}/bindings/python/tests/fdbcli_tests.py
            ${CMAKE_BINARY_DIR}
            @CLUSTER_FILE@
            5
            )
  if (TARGET external_client) # external_client copies fdb_c to bindings/c/libfdb_c_external.so
    add_fdbclient_test(
    NAME single_process_external_client_fdbcli_tests
    COMMAND ${CMAKE_SOURCE_DIR}/bindings/python/tests/fdbcli_tests.py
            ${CMAKE_BINARY_DIR}
            @CLUSTER_FILE@
            --external-client-library ${CMAKE_BINARY_DIR}/bindings/c/libfdb_c_external.so
            )
    add_fdbclient_test(
    NAME multi_process_external_client_fdbcli_tests
    PROCESS_NUMBER 5
    COMMAND ${CMAKE_SOURCE_DIR}/bindings/python/tests/fdbcli_tests.py
            ${CMAKE_BINARY_DIR}
            @CLUSTER_FILE@
            5
            --external-client-library ${CMAKE_BINARY_DIR}/bindings/c/libfdb_c_external.so
            )
  endif()
endif()
