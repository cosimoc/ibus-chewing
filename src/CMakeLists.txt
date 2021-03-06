#==================================================================
# Variable definition
#
IF(NOT DEFINED LIB_INSTALL_DIR)
    SET(LIB_INSTALL_DIR "lib${IS_64}")
ENDIF()
SET_COMPILE_ENV(LIB_INSTALL_DIR  "${LIB_INSTALL_DIR}"
    DISPLAY STRING "Chewing library directory")

##== Location of library include files
INCLUDE_DIRECTORIES(
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_BINARY_DIR}
    )

IF("${CMAKE_BUILD_TYPE}" STREQUAL "Debug")
    ADD_DEFINITIONS("-DVERBOSE_LEVEL=DEBUG")
ELSE()
    ADD_DEFINITIONS("-DVERBOSE_LEVEL=MSG")
ENDIF()

SET(IBUS_ENGINE_CHEWING ibus-engine-chewing)
SET(IBUS_SETUP_CHEWING ibus-setup-chewing)

#==================================================================
# Dependencies
#

SET(IBUS_CHEWING_DEPENDENCIES CHEWING GLIB2 GTK2 IBUS X11)

IF(GCONF2_SUPPORT)
    # Schemas
    INCLUDE(ManageGConf)
    LIST(APPEND IBUS_CHEWING_DEPENDENCIES "GCONF2")
    MANAGE_GCONF_SCHEMAS()
ENDIF()

SET(IBUS_CHEWING_DEPENDENCIES "${IBUS_CHEWING_DEPENDENCIES}"
    CACHE INTERAL "IBUS_CHEWING_DEPENDENCIES"
    )

FOREACH(_d ${IBUS_CHEWING_DEPENDENCIES})
    INCLUDE_DIRECTORIES(${${_d}_INCLUDE_DIRS})
#    STRING_APPEND(CMAKE_C_FLAGS "${${_d}_CFLAGS}" " ")
ENDFOREACH(_d ${IBUS_CHEWING_DEPENDENCIES})


#==================================================================
# Sources
#

##== Gob Generate Start ==
SET(IBUS_CHEWING_ENGINE_SOURCE_INCLUDED_C_FILES
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusChewingEngine-signal.c
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusChewingEngine-input-events.c
    )

SET(GOB2_OUTPUT_DIR "${CMAKE_CURRENT_BINARY_DIR}")
MACRO(GOB_GENERATE name gobFile)
    SET(GOB_GENERATED_C_${name}
	${GOB2_OUTPUT_DIR}/${name}.c)
    SET(GOB_GENERATED_${name}
	${GOB_GENERATED_C_${name}}
	"${GOB2_OUTPUT_DIR}/${name}.h"
	"${GOB2_OUTPUT_DIR}/${name}-private.h"
	CACHE INTERNAL "GOB_GENERATED_${name}"
	)

    ADD_CUSTOM_COMMAND(OUTPUT ${GOB_GENERATED_${name}}
	COMMAND ${GOB2_EXECUTABLE} "${gobFile}"
	-o "${GOB2_OUTPUT_DIR}/"
	DEPENDS ${gobFile} ${ARGN}
	)
ENDMACRO(GOB_GENERATE name gobFile)


GOB_GENERATE(ibus-chewing-engine
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusChewingEngine.gob
    ${IBUS_CHEWING_ENGINE_SOURCE_INCLUDED_C_FILES}
    )
GOB_GENERATE(maker-dialog
    ${CMAKE_CURRENT_SOURCE_DIR}/MakerDialog.gob
    )

ADD_CUSTOM_TARGET(gob2
    DEPENDS ${GOB_GENERATED_ibus-chewing-engine}
    ${GOB_GENERATED_maker-dialog}
    COMMENT "Preprocess with gob2."
    )


SET_SOURCE_FILES_PROPERTIES(${GOB_GENERATED_ibus-chewing-engine}
    ${GOB_GENERATED_maker-dialog} PROPERTIES GENERATED true
    )

SET(COMMON_SOURCE_C_FILES
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusChewingApplier.c
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusChewingLookupTable.c
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusChewingProperties.c
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusConfigBackend.c
    ${CMAKE_CURRENT_SOURCE_DIR}/MakerDialogBackend.c
    ${CMAKE_CURRENT_SOURCE_DIR}/MakerDialogProperty.c
    ${CMAKE_CURRENT_SOURCE_DIR}/MakerDialogUtil.c
    )

SET(COMMON_SOURCE_FILES
    ${COMMON_SOURCE_C_FILES}
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusChewingLookupTable.h
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusChewingProperties.h
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusConfigBackend.h
    ${CMAKE_CURRENT_SOURCE_DIR}/MakerDialogBackend.h
    ${CMAKE_CURRENT_SOURCE_DIR}/MakerDialogProperty.h
    ${CMAKE_CURRENT_SOURCE_DIR}/MakerDialogPropertySpec.h
    ${CMAKE_CURRENT_SOURCE_DIR}/MakerDialogUtil.h
    ${GOB2_OUTPUT_DIR}/ibus-chewing-engine.h
    CACHE INTERNAL "COMMON_SOURCE_FILES"
    )

FILE(GLOB poFiles RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}/../po" "../po/*.po")
STRING(REPLACE ".po" ""  TRANSLATED "${poFiles}")
M_MSG(${M_INFO1} "TRANSLATED=${TRANSLATED}")

#==================================================================
# Schemas
#
SET(BACKEND_SOURCE_FILES "")

IF(GSETTINGS_SUPPORT)
    LIST(APPEND BACKEND_SOURCE_FILES
	"${CMAKE_SOURCE_DIR}/src/GSettingsBackend.c"
	"${CMAKE_SOURCE_DIR}/src/GSettingsBackend.h"
	)
ENDIF()

IF(GCONF2_SUPPORT)
    LIST(APPEND BACKEND_SOURCE_FILES
	"${CMAKE_SOURCE_DIR}/src/GConf2Backend.c"
	"${CMAKE_SOURCE_DIR}/src/GConf2Backend.h"
	)
ENDIF(GCONF2_SUPPORT)

SET(BACKEND_SOURCE_FILES "${BACKEND_SOURCE_FILES}" CACHE INTERNAL "Backend source files")

SET(COMMON_LIBRARIES
    ${CHEWING_LIBRARIES}
    ${IBUS_LIBRARIES}
    ${GLIB2_LIBRARIES}
    )

SET(BACKEND_LIBRARIES "")

##== GSettings support==
IF(GSETTINGS_SUPPORT)
    SET(PROJECT_GSCHEMA_XML ${PROJECT_SCHEMA_ID}.gschema.xml
	    CACHE INTERNAL "PROJECT_GSCHEMA_XML")
    SET(GSETTINGS_SCHEMAS_DIR "${DATA_DIR}/glib-2.0/schemas")

    ADD_EXECUTABLE(generate-gsettings-schemas
	${COMMON_SOURCE_FILES}
	${CMAKE_CURRENT_SOURCE_DIR}/GSettingsBackend.c
	${CMAKE_CURRENT_SOURCE_DIR}/GSettingsBackend.h
	${CMAKE_CURRENT_SOURCE_DIR}/generate-gsettings-schemas.c
	)

    TARGET_LINK_LIBRARIES(generate-gsettings-schemas
	${COMMON_LIBRARIES}
	)

    INSTALL(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_GSCHEMA_XML}
	DESTINATION "${GSETTINGS_SCHEMAS_DIR}"
	)

    ADD_CUSTOM_TARGET(schemas_gsettings ALL
	COMMAND G_MESSAGES_DEBUG=all "${CMAKE_CURRENT_BINARY_DIR}/generate-gsettings-schemas" -v 5 "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_GSCHEMA_XML}"
	COMMAND G_MESSAGES_DEBUG=all glib-compile-schemas "${CMAKE_CURRENT_BINARY_DIR}"
	COMMENT "Generating gsettings schemas ${PROJECT_GSCHEMA_XML}"
	VERBATIM
	)

    ADD_DEPENDENCIES(schemas_gsettings
	generate-gsettings-schemas
	)

    ADD_CUSTOM_TARGET(install_schemas_gsettings
	COMMAND install "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_GSCHEMA_XML}"  "${GSETTINGS_SCHEMAS_DIR}/${PROJECT_GSCHEMA_XML}"
	COMMAND glib-compile-schemas "${GSETTINGS_SCHEMAS_DIR}"
	COMMENT "install_schemas_gsettings"
	)

    ADD_CUSTOM_TARGET(uninstall_schemas_gsettings
	COMMAND rm -f "${GSETTINGS_SCHEMAS_DIR}/${PROJECT_GSCHEMA_XML}"
	COMMAND glib-compile-schemas "${GSETTINGS_SCHEMAS_DIR}"
	COMMENT "install_schemas_gsettings"
	)
ENDIF()

##== GConf2 support==
IF(GCONF2_SUPPORT)
    SET(PROJECT_SCHEMAS ${PROJECT_NAME}.schemas
	CACHE INTERNAL "PROJECT_SCHEMAS")
    SET(GCONF2_SCHEMAS_DIR "${SYSCONF_DIR}/gconf/schemas")

    SET(BACKEND_LIBRARIES ${GCONF_LIBRARIES})

    ADD_EXECUTABLE(generate-gconf-schemas
	${COMMON_SOURCE_FILES}
	${CMAKE_CURRENT_SOURCE_DIR}/GConf2Backend.c
	${CMAKE_CURRENT_SOURCE_DIR}/GConf2Backend.h
	${CMAKE_CURRENT_SOURCE_DIR}/generate-gconf-schemas.c
	)

    LIST(APPEND COMMON_LIBRARIES ${GCONF2_LIBRARIES})

    TARGET_LINK_LIBRARIES(generate-gconf-schemas
	${COMMON_LIBRARIES} ${BACKEND_LIBRARIES}
	)

    ## ManageGConf is already handle install

    ADD_CUSTOM_TARGET(schemas_gconf2 ALL
	COMMAND G_MESSAGES_DEBUG=all "${CMAKE_CURRENT_BINARY_DIR}/generate-gconf-schemas" -v 5 -l
	"C;${TRANSLATED}" "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_SCHEMAS}"
	COMMENT "Generating gconf-schemas ${PROJECT_SCHEMAS}"
	VERBATIM
	)

    ADD_DEPENDENCIES(schemas_gconf2
	generate-gconf-schemas
	)

    ADD_DEPENDENCIES(generate-gconf-schemas
	translations
	)

ENDIF()

SET(BACKEND_LIBRARIES "${BACKEND_LIBRARIES}" CACHE INTERNAL "Backend Libraries")

#==================================================================
# Executables
#

SET(GUI_CORE_SOURCE_FILES ${COMMON_SOURCE_FILES}
    ${BACKEND_SOURCE_FILES}
    ${GOB_GENERATED_maker-dialog}
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusConfigBackend.c
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusConfigBackend.h
    ${CMAKE_CURRENT_SOURCE_DIR}/MakerDialogWidget.c
    ${CMAKE_CURRENT_SOURCE_DIR}/MakerDialogWidget.h
    CACHE INTERNAL "GUI core source files"
    )


IF(GSETTINGS_SUPPORT)
    LIST(APPEND GUI_CORE_SOURCE_FILES
	${CMAKE_CURRENT_SOURCE_DIR}/GSettingsBackend.c
	${CMAKE_CURRENT_SOURCE_DIR}/GSettingsBackend.h
	)
ENDIF()

IF(GCONF2_SUPPORT)
    LIST(APPEND GUI_CORE_SOURCE_FILES
	${CMAKE_CURRENT_SOURCE_DIR}/GConf2Backend.c
	${CMAKE_CURRENT_SOURCE_DIR}/GConf2Backend.h
	)
ENDIF()

##== ibus-setup-chewing Start ==
ADD_EXECUTABLE(${IBUS_SETUP_CHEWING}
    ${GUI_CORE_SOURCE_FILES}
    ${CMAKE_CURRENT_SOURCE_DIR}/ibus-setup-chewing.c
    )

TARGET_LINK_LIBRARIES(${IBUS_SETUP_CHEWING}
    ${COMMON_LIBRARIES}
    ${GTK2_LIBRARIES}
    )

MANAGE_FILE_INSTALL(TARGETS ${IBUS_SETUP_CHEWING} ARGS
    LIBEXEC DESTINATION ${LIBEXEC_DIR}
    LIBRARY DESTINATION ${LIB_INSTALL_DIR}
    )

##== ibus-engine-chewing Start ==
SET(IBUS_ENGINE_CHEWING_LIBRARIES ${COMMON_LIBRARIES}
    ${GTK2_LIBRARIES}
    ${X11_LIBRARIES}
    CACHE INTERNAL "IBUS_ENGINE_CHEWING_LIBRARIES"
    )

ADD_EXECUTABLE(${IBUS_ENGINE_CHEWING}
    ${GUI_CORE_SOURCE_FILES}
    ${GOB_GENERATED_ibus-chewing-engine}
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusChewingLookupTable.c
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusChewingLookupTable.h
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusChewingPreEdit.c
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusChewingPreEdit.h
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusChewingSystray.c
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusChewingSystray.h
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusChewingUtil.c
    ${CMAKE_CURRENT_SOURCE_DIR}/IBusChewingUtil.h
    ${CMAKE_CURRENT_SOURCE_DIR}/main.c
    )

TARGET_LINK_LIBRARIES(${IBUS_ENGINE_CHEWING}
    ${IBUS_ENGINE_CHEWING_LIBRARIES}
    )


MANAGE_FILE_INSTALL(TARGETS ${IBUS_ENGINE_CHEWING} ARGS
    LIBEXEC DESTINATION ${LIBEXEC_DIR}
    LIBRARY DESTINATION ${LIB_INSTALL_DIR}
    )

SET(COMMON_LIBRARIES "${COMMON_LIBRARIES}" CACHE INTERNAL "Common Libraries")
