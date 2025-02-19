# SPDX-FileCopyrightText: KDE Contributors
# SPDX-License-Identifier: BSD-2-Clause

cmake_minimum_required(VERSION 3.16)

set(KF_VERSION "5.240.0") # handled by release scripts
set(KF_DEP_VERSION "5.240.0") # handled by release scripts
project(KRunner VERSION ${KF_VERSION})

# ECM setup
include(FeatureSummary)
find_package(ECM 5.240.0  NO_MODULE)
set_package_properties(ECM PROPERTIES TYPE REQUIRED DESCRIPTION "Extra CMake Modules." URL "https://commits.kde.org/extra-cmake-modules")
feature_summary(WHAT REQUIRED_PACKAGES_NOT_FOUND FATAL_ON_MISSING_REQUIRED_PACKAGES)

set(CMAKE_MODULE_PATH ${ECM_MODULE_PATH})

include(KDEInstallDirs)
include(KDECMakeSettings)
include(KDEFrameworkCompilerSettings NO_POLICY_SCOPE)

include(ECMGenerateExportHeader)
include(ECMSetupVersion)
include(ECMGenerateHeaders)
include(ECMAddQch)
include(CMakePackageConfigHelpers)
include(KDEPackageAppTemplates)
include(ECMQtDeclareLoggingCategory)
include(ECMSetupQtPluginMacroNames)
include(ECMDeprecationSettings)
include(KDEGitCommitHooks)


set(EXCLUDE_DEPRECATED_BEFORE_AND_AT 0 CACHE STRING "Control the range of deprecated API excluded from the build [default=0].")

option(BUILD_QCH "Build API documentation in QCH format (for e.g. Qt Assistant, Qt Creator & KDevelop)" OFF)
add_feature_info(QCH ${BUILD_QCH} "API documentation in QCH format (for e.g. Qt Assistant, Qt Creator & KDevelop)")

ecm_setup_version(PROJECT
    VARIABLE_PREFIX KRUNNER
    VERSION_HEADER "${CMAKE_CURRENT_BINARY_DIR}/krunner_version.h"
    PACKAGE_VERSION_FILE "${CMAKE_CURRENT_BINARY_DIR}/KF6RunnerConfigVersion.cmake"
)

# Dependencies
set(REQUIRED_QT_VERSION 6.4.0)

find_package(Qt${QT_MAJOR_VERSION} ${REQUIRED_QT_VERSION} NO_MODULE REQUIRED Gui Widgets Quick)

ecm_set_disabled_deprecation_versions(
    QT ${REQUIRED_QT_VERSION}
    KF 5.95.0
    PLASMA 5.0.0  # needed because we use Plasma::Package in the API
    KSERVICE 5.89.0  # needed because we use KPluginInfo, KServiceTypeTrader in the API
)

find_package(KF6Config ${KF_DEP_VERSION} REQUIRED)
find_package(KF6CoreAddons ${KF_DEP_VERSION} REQUIRED)
find_package(KF6I18n ${KF_DEP_VERSION} REQUIRED)
find_package(KF6ThreadWeaver ${KF_DEP_VERSION} REQUIRED)
if(NOT EXCLUDE_DEPRECATED_BEFORE_AND_AT STREQUAL "CURRENT" AND
   EXCLUDE_DEPRECATED_BEFORE_AND_AT VERSION_LESS 5.91.0)
    find_package(KF6KIO ${KF_DEP_VERSION} REQUIRED)
    find_package(KF6Service ${KF_DEP_VERSION} REQUIRED)
    find_package(KF6Plasma ${KF_DEP_VERSION} REQUIRED)
else()
    set(KF6Plasma_FOUND 0)
endif()

find_package(KF6Activities ${KF_DEP_VERSION})
set_package_properties(KF6Activities PROPERTIES
        DESCRIPTION "Library for Activities"
        PURPOSE "Required for activities support (optional)"
        TYPE OPTIONAL
        )
if(KF6Activities_FOUND)
    set(HAVE_KACTIVITIES TRUE)
endif()

# TODO KF6 remove
ecm_setup_qtplugin_macro_names(
    JSON_NONE
        "K_EXPORT_PLASMA_RUNNER"
        "K_EXPORT_RUNNER_CONFIG"
    JSON_ARG2
        "K_EXPORT_PLASMA_RUNNER_WITH_JSON"
    CONFIG_CODE_VARIABLE
        PACKAGE_SETUP_AUTOMOC_VARIABLES
)

# Subdirectories
add_subdirectory(src)
if (BUILD_TESTING)
    add_subdirectory(autotests)
endif()
add_subdirectory(templates)

# Create a Config.cmake and a ConfigVersion.cmake file and install them
set(CMAKECONFIG_INSTALL_DIR "${KDE_INSTALL_CMAKEPACKAGEDIR}/KF6Runner")

if (BUILD_QCH)
    ecm_install_qch_export(
        TARGETS KF6Runner_QCH
        FILE KF6RunnerQchTargets.cmake
        DESTINATION "${CMAKECONFIG_INSTALL_DIR}"
        COMPONENT Devel
    )
    set(PACKAGE_INCLUDE_QCHTARGETS "include(\"\${CMAKE_CURRENT_LIST_DIR}/KF6RunnerQchTargets.cmake\")")
endif()

configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/KF6RunnerConfig.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/KF6RunnerConfig.cmake"
    INSTALL_DESTINATION "${CMAKECONFIG_INSTALL_DIR}"
)

install(FILES
            "${CMAKE_CURRENT_BINARY_DIR}/KF6RunnerConfig.cmake"
            "${CMAKE_CURRENT_BINARY_DIR}/KF6RunnerConfigVersion.cmake"
            "${CMAKE_CURRENT_SOURCE_DIR}/KF6KRunnerMacros.cmake"
        DESTINATION "${CMAKECONFIG_INSTALL_DIR}"
        COMPONENT Devel)

install(EXPORT KF6RunnerTargets
        DESTINATION "${CMAKECONFIG_INSTALL_DIR}"
        FILE KF6RunnerTargets.cmake
        NAMESPACE KF6::)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/krunner_version.h"
        DESTINATION "${KDE_INSTALL_INCLUDEDIR_KF}/KRunner"
        COMPONENT Devel)

feature_summary(WHAT ALL FATAL_ON_MISSING_REQUIRED_PACKAGES)

kde_configure_git_pre_commit_hook(CHECKS CLANG_FORMAT)
