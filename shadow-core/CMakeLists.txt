#设置模块名称
set(SHADOW_CORE_TARGET "shadow-core")

#
# Source files.
#
aux_source_directory(. SHADOW_CORE_DIR)

set(SHADOW_CORE_SRCS ${SHADOW_CORE_DIR})
# linux
# set(SHADOW_CORE_SRCS_LINUX
#   cefsimple_linux.cc
#   simple_handler_linux.cc
#   )
# macos
# set(SHADOW_CORE_SRCS_MACOSX
#   cefsimple_mac.mm
#   simple_handler_mac.mm
#   )
# windows
# set(SHADOW_CORE_SRCS_WINDOWS
#   shadow-core.rc
#   cefsimple_win.cpp
#   resource.h
#   simple_handler_win.cpp
# 	)

APPEND_PLATFORM_SOURCES(SHADOW_CORE_SRCS)
source_group(shadow-core FILES ${SHADOW_CORE_SRCS})

set(SHADOW_CORE_SRCS
  ${SHADOW_CORE_SRCS}
  )

# shadow-core helper sources.
# set(SHADOW_HELPER_SRCS_MACOSX
#   process_helper_mac.cc
# 	)

APPEND_PLATFORM_SOURCES(SHADOW_HELPER_SRCS)
source_group(shadow-core FILES ${SHADOW_HELPER_SRCS})

# shadow-core resources.
set(SHADOW_RESOURCES_MAC_SRCS_MACOSX
  mac/Info.plist
  mac/shadow-core.icns
  )
APPEND_PLATFORM_SOURCES(SHADOW_RESOURCES_MAC_SRCS)
source_group(shadow-core\\\\mac FILES ${SHADOW_RESOURCES_MAC_SRCS})

set(SHADOW_RESOURCES_MAC_ENGLISH_LPROJ_SRCS_MACOSX
  mac/English.lproj/InfoPlist.strings
  mac/English.lproj/MainMenu.xib
  )
APPEND_PLATFORM_SOURCES(SHADOW_RESOURCES_MAC_ENGLISH_LPROJ_SRCS)
source_group(shadow-core\\\\mac\\\\English.lproj FILES ${SHADOW_RESOURCES_MAC_ENGLISH_LPROJ_SRCS})

set(SHADOW_RESOURCES_SRCS
  ${SHADOW_RESOURCES_MAC_SRCS}
  ${SHADOW_RESOURCES_MAC_ENGLISH_LPROJ_SRCS}
)


#
# Shared configuration.
#


if(OS_MACOSX)
  set(CEF_HELPER_TARGET "shadow-core_Helper")
  set(CEF_HELPER_OUTPUT_NAME "shadow-core Helper")
else()
  # Logical target used to link the libcef library.
  ADD_LOGICAL_TARGET("libcef_lib" "${CEF_LIB_DEBUG}" "${CEF_LIB_RELEASE}")
endif()

# Determine the target output directory.
SET_CEF_TARGET_OUT_DIR()


#
# Linux configuration.
#

if(OS_LINUX)
  # Executable target.
  add_executable(${SHADOW_CORE_TARGET} ${SHADOW_CORE_SRCS})
  SET_EXECUTABLE_TARGET_PROPERTIES(${SHADOW_CORE_TARGET})
  add_dependencies(${SHADOW_CORE_TARGET} libcef_dll_wrapper)
  target_link_libraries(${SHADOW_CORE_TARGET} libcef_lib libcef_dll_wrapper ${CEF_STANDARD_LIBS})

  # Set rpath so that libraries can be placed next to the executable.
  set_target_properties(${SHADOW_CORE_TARGET} PROPERTIES INSTALL_RPATH "$ORIGIN")
  set_target_properties(${SHADOW_CORE_TARGET} PROPERTIES BUILD_WITH_INSTALL_RPATH TRUE)
  set_target_properties(${SHADOW_CORE_TARGET} PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${CEF_TARGET_OUT_DIR})

  # Copy binary and resource files to the target output directory.
  COPY_FILES("${SHADOW_CORE_TARGET}" "${CEF_BINARY_FILES}" "${CEF_BINARY_DIR}" "${CEF_TARGET_OUT_DIR}")
  COPY_FILES("${SHADOW_CORE_TARGET}" "${CEF_RESOURCE_FILES}" "${CEF_RESOURCE_DIR}" "${CEF_TARGET_OUT_DIR}")
  if (EXISTS "${CEF_BINARY_DIR}/libminigbm.so")
    COPY_FILES("${SHADOW_CORE_TARGET}" "libminigbm.so" "${CEF_BINARY_DIR}" "${CEF_TARGET_OUT_DIR}")
  endif()

  # Set SUID permissions on the chrome-sandbox target.
  SET_LINUX_SUID_PERMISSIONS("${SHADOW_CORE_TARGET}" "${CEF_TARGET_OUT_DIR}/chrome-sandbox")
endif()


#
# Mac OS X configuration.
#

if(OS_MACOSX)
  option(OPTION_USE_ARC "Build with ARC (automatic Reference Counting) on macOS." ON)
  if(OPTION_USE_ARC)
    list(APPEND CEF_COMPILER_FLAGS
      -fobjc-arc
      )
    set_target_properties(${target} PROPERTIES
      CLANG_ENABLE_OBJC_ARC "YES"
      )
  endif()

  # Output path for the main app bundle.
  set(CEF_APP "${CEF_TARGET_OUT_DIR}/${SHADOW_CORE_TARGET}.app")

  # Variables referenced from the main Info.plist file.
  set(EXECUTABLE_NAME "${SHADOW_CORE_TARGET}")
  set(PRODUCT_NAME "${SHADOW_CORE_TARGET}")

  if(USE_SANDBOX)
    # Logical target used to link the cef_sandbox library.
    ADD_LOGICAL_TARGET("cef_sandbox_lib" "${CEF_SANDBOX_LIB_DEBUG}" "${CEF_SANDBOX_LIB_RELEASE}")
  endif()

  # Main app bundle target.
  add_executable(${SHADOW_CORE_TARGET} MACOSX_BUNDLE ${SHADOW_RESOURCES_SRCS} ${SHADOW_CORE_SRCS})
  SET_EXECUTABLE_TARGET_PROPERTIES(${SHADOW_CORE_TARGET})
  add_dependencies(${SHADOW_CORE_TARGET} libcef_dll_wrapper)
  target_link_libraries(${SHADOW_CORE_TARGET} libcef_dll_wrapper ${CEF_STANDARD_LIBS})
  set_target_properties(${SHADOW_CORE_TARGET} PROPERTIES
    MACOSX_BUNDLE_INFO_PLIST ${CMAKE_CURRENT_SOURCE_DIR}/mac/Info.plist
    )

  # Copy the CEF framework into the Frameworks directory.
  add_custom_command(
    TARGET ${SHADOW_CORE_TARGET}
    POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
            "${CEF_BINARY_DIR}/Chromium Embedded Framework.framework"
            "${CEF_APP}/Contents/Frameworks/Chromium Embedded Framework.framework"
    VERBATIM
    )

  # Create the multiple Helper app bundle targets.
  foreach(_suffix_list ${CEF_HELPER_APP_SUFFIXES})
    # Convert to a list and extract the suffix values.
    string(REPLACE ":" ";" _suffix_list ${_suffix_list})
    list(GET _suffix_list 0 _name_suffix)
    list(GET _suffix_list 1 _target_suffix)
    list(GET _suffix_list 2 _plist_suffix)

    # Define Helper target and output names.
    set(_helper_target "${CEF_HELPER_TARGET}${_target_suffix}")
    set(_helper_output_name "${CEF_HELPER_OUTPUT_NAME}${_name_suffix}")

    # Create Helper-specific variants of the helper-Info.plist file. Do this
    # manually because the configure_file command (which is executed as part of
    # MACOSX_BUNDLE_INFO_PLIST) uses global env variables and would insert the
    # wrong values with multiple targets.
    set(_helper_info_plist "${CMAKE_CURRENT_BINARY_DIR}/helper-Info${_target_suffix}.plist")
    file(READ "${CMAKE_CURRENT_SOURCE_DIR}/mac/helper-Info.plist" _plist_contents)
    string(REPLACE "\${EXECUTABLE_NAME}" "${_helper_output_name}" _plist_contents ${_plist_contents})
    string(REPLACE "\${PRODUCT_NAME}" "${_helper_output_name}" _plist_contents ${_plist_contents})
    string(REPLACE "\${BUNDLE_ID_SUFFIX}" "${_plist_suffix}" _plist_contents ${_plist_contents})
    file(WRITE ${_helper_info_plist} ${_plist_contents})

    # Create Helper executable target.
    add_executable(${_helper_target} MACOSX_BUNDLE ${SHADOW_HELPER_SRCS})
    SET_EXECUTABLE_TARGET_PROPERTIES(${_helper_target})
    add_dependencies(${_helper_target} libcef_dll_wrapper)
    target_link_libraries(${_helper_target} libcef_dll_wrapper ${CEF_STANDARD_LIBS})
    set_target_properties(${_helper_target} PROPERTIES
      MACOSX_BUNDLE_INFO_PLIST ${_helper_info_plist}
      OUTPUT_NAME ${_helper_output_name}
      )

    if(USE_SANDBOX)
      target_link_libraries(${_helper_target} cef_sandbox_lib)
    endif()

    # Add the Helper as a dependency of the main executable target.
    add_dependencies(${SHADOW_CORE_TARGET} "${_helper_target}")

    # Copy the Helper app bundle into the Frameworks directory.
    add_custom_command(
      TARGET ${SHADOW_CORE_TARGET}
      POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E copy_directory
              "${CEF_TARGET_OUT_DIR}/${_helper_output_name}.app"
              "${CEF_APP}/Contents/Frameworks/${_helper_output_name}.app"
      VERBATIM
      )
  endforeach()

  # Manually process and copy over resource files.
  # The Xcode generator can support this via the set_target_properties RESOURCE
  # directive but that doesn't properly handle nested resource directories.
  # Remove these prefixes from input file paths.
  set(PREFIXES "mac/")
  COPY_MACOSX_RESOURCES("${SHADOW_RESOURCES_SRCS}" "${PREFIXES}" "${SHADOW_CORE_TARGET}" "${CMAKE_CURRENT_SOURCE_DIR}" "${CEF_APP}")
endif()


#
# Windows configuration.
#

if(OS_WINDOWS)
  # Executable target.
  add_executable(${SHADOW_CORE_TARGET} WIN32 ${SHADOW_CORE_SRCS})
  add_dependencies(${SHADOW_CORE_TARGET} libcef_dll_wrapper)
  SET_EXECUTABLE_TARGET_PROPERTIES(${SHADOW_CORE_TARGET})
  target_link_libraries(${SHADOW_CORE_TARGET} libcef_lib libcef_dll_wrapper ${CEF_STANDARD_LIBS})

  if(USE_SANDBOX)
    # Logical target used to link the cef_sandbox library.
    ADD_LOGICAL_TARGET("cef_sandbox_lib" "${CEF_SANDBOX_LIB_DEBUG}" "${CEF_SANDBOX_LIB_RELEASE}")
    target_link_libraries(${SHADOW_CORE_TARGET} cef_sandbox_lib ${CEF_SANDBOX_STANDARD_LIBS})
  endif()

  # Add the custom manifest files to the executable.
  ADD_WINDOWS_MANIFEST("${CMAKE_CURRENT_SOURCE_DIR}" "${SHADOW_CORE_TARGET}" "exe")

  # Copy binary and resource files to the target output directory.
  COPY_FILES("${SHADOW_CORE_TARGET}" "${CEF_BINARY_FILES}" "${CEF_BINARY_DIR}" "${CEF_TARGET_OUT_DIR}")
  COPY_FILES("${SHADOW_CORE_TARGET}" "${CEF_RESOURCE_FILES}" "${CEF_RESOURCE_DIR}" "${CEF_TARGET_OUT_DIR}")
endif()
