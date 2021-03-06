include(ECMAddAppIcon)

find_package(Qt5 REQUIRED COMPONENTS Widgets)

add_subdirectory(updater)

set(client_UI_SRCS
    aboutdialog.ui
    accountsettings.ui
    folderwizardsourcepage.ui
    folderwizardtargetpage.ui
    generalsettings.ui
    ignorelisteditor.ui
    networksettings.ui
    protocolwidget.ui
    issueswidget.ui
    activitywidget.ui
    settingsdialog.ui
    sharedialog.ui
    sharelinkwidget.ui
    shareusergroupwidget.ui
    shareuserline.ui
    sslerrordialog.ui
    addcertificatedialog.ui
    proxyauthdialog.ui
    notificationwidget.ui
    logbrowser.ui
    wizard/owncloudadvancedsetuppage.ui
    wizard/owncloudhttpcredspage.ui
    wizard/owncloudoauthcredspage.ui
    wizard/owncloudsetuppage.ui
    wizard/owncloudwizardresultpage.ui
)

set(client_SRCS
    aboutdialog.cpp
    accountmanager.cpp
    accountsettings.cpp
    application.cpp
    clientproxy.cpp
    connectionvalidator.cpp
    folder.cpp
    folderman.cpp
    folderstatusmodel.cpp
    folderstatusdelegate.cpp
    folderwatcher.cpp
    folderwizard.cpp
    generalsettings.cpp
    ignorelisteditor.cpp
    lockwatcher.cpp
    logbrowser.cpp
    networksettings.cpp
    ocsjob.cpp
    ocssharejob.cpp
    ocsshareejob.cpp
    openfilemanager.cpp
    owncloudgui.cpp
    owncloudsetupwizard.cpp
    protocolwidget.cpp
    issueswidget.cpp
    activitydata.cpp
    activitylistmodel.cpp
    activitywidget.cpp
    activityitemdelegate.cpp
    selectivesyncdialog.cpp
    settingsdialog.cpp
    sharedialog.cpp
    sharelinkwidget.cpp
    sharemanager.cpp
    shareusergroupwidget.cpp
    sharee.cpp
    socketapi.cpp
    sslbutton.cpp
    sslerrordialog.cpp
    syncrunfilelog.cpp
    systray.cpp
    thumbnailjob.cpp
    quotainfo.cpp
    accountstate.cpp
    addcertificatedialog.cpp
    authenticationdialog.cpp
    proxyauthhandler.cpp
    proxyauthdialog.cpp
    tooltipupdater.cpp
    notificationwidget.cpp
    notificationconfirmjob.cpp
    servernotificationhandler.cpp
    guiutility.cpp
    elidedlabel.cpp
    creds/httpcredentialsgui.cpp
    wizard/postfixlineedit.cpp
    wizard/abstractcredswizardpage.cpp
    wizard/owncloudadvancedsetuppage.cpp
    wizard/owncloudhttpcredspage.cpp
    wizard/owncloudoauthcredspage.cpp
    wizard/owncloudsetuppage.cpp
    wizard/owncloudwizardcommon.cpp
    wizard/owncloudwizard.cpp
    wizard/owncloudwizardresultpage.cpp
)

set(3rdparty_SRC
    ../3rdparty/QProgressIndicator/QProgressIndicator.cpp
    ../3rdparty/qtlockedfile/qtlockedfile.cpp
    ../3rdparty/qtsingleapplication/qtlocalpeer.cpp
    ../3rdparty/qtsingleapplication/qtsingleapplication.cpp
    ../3rdparty/qtsingleapplication/qtsinglecoreapplication.cpp
   )

if(NOT WIN32)
   list(APPEND 3rdparty_SRC ../3rdparty/qtlockedfile/qtlockedfile_unix.cpp)
else()
   list(APPEND 3rdparty_SRC ../3rdparty/qtlockedfile/qtlockedfile_win.cpp )
endif()

set( final_src
    ${MIRALL_RC_SRC}
    ${client_SRCS}
    ${client_UI_SRCS}
    ${3rdparty_SRC}
)

if(TARGET qt5keychain)
    list(APPEND libsync_LINK_TARGETS qt5keychain)
endif()

add_library(owncloudCore STATIC ${final_src})
set_target_properties(owncloudCore PROPERTIES AUTOUIC ON AUTORCC ON)
target_link_libraries(owncloudCore PUBLIC Qt5::Widgets Qt5::Network Qt5::Xml
                                          ${synclib_NAME})

target_include_directories(owncloudCore PUBLIC
    ${CMAKE_SOURCE_DIR}/src/3rdparty/QProgressIndicator
    ${CMAKE_SOURCE_DIR}/src/3rdparty/qtlockedfile
    ${CMAKE_SOURCE_DIR}/src/3rdparty/qtsingleapplication
    ${CMAKE_CURRENT_BINARY_DIR}
)


target_sources(owncloudCore PRIVATE
    updater/ocupdater.cpp
    updater/updateinfo.cpp
    updater/updater.cpp
)

IF( APPLE )
    find_package(Qt5 COMPONENTS MacExtras)
    target_link_libraries(owncloudCore PUBLIC Qt5::MacExtras)

    target_sources(owncloudCore PRIVATE
            cocoainitializer_mac.mm
            socketapisocket_mac.mm
            systray.mm
            settingsdialog_mac.mm

            folderwatcher_mac.cpp)

    if(SPARKLE_FOUND)
        # Define this, we need to check in updater.cpp
        target_compile_definitions(owncloudCore PUBLIC HAVE_SPARKLE)
        target_sources(owncloudCore PRIVATE
            updater/sparkleupdater_mac.mm
            updater/sparkleupdater.h)
        target_link_libraries(owncloudCore PRIVATE ${SPARKLE_LIBRARY})
   endif()
elseif( WIN32 )
    target_sources(owncloudCore PRIVATE
            folderwatcher_win.cpp
            navigationpanehelper.cpp)
elseif(UNIX AND NOT APPLE )
    ## handle DBUS for Fdo notifications
    find_package(Qt5 COMPONENTS DBus)
    if (TARGET Qt5::DBus)
        target_link_libraries(owncloudCore PUBLIC Qt5::DBus)
        target_compile_definitions(owncloudCore PUBLIC "USE_FDO_NOTIFICATIONS")
    endif()
    target_sources(owncloudCore PRIVATE folderwatcher_linux.cpp)
endif()


if(WITH_CRASHREPORTER)
    target_link_libraries(owncloudCore PUBLIC crashreporter-handler)

    if(UNIX AND NOT MAC)
        find_package(Threads REQUIRED)
        target_link_libraries(owncloudCore PUBLIC ${CMAKE_THREAD_LIBS_INIT})
    endif()
endif()

add_executable(${APPLICATION_EXECUTABLE} main.cpp)
set_target_properties(${APPLICATION_EXECUTABLE} PROPERTIES AUTOUIC ON AUTORCC ON)
target_link_libraries( ${APPLICATION_EXECUTABLE} owncloudCore )


find_package(Qt5LinguistTools)
if(Qt5LinguistTools_FOUND)
    qt5_add_translation(client_I18N ${TRANSLATIONS})
    target_sources(${APPLICATION_EXECUTABLE} PRIVATE ${client_I18N})
endif()

#TODO Move resources files
target_sources(${APPLICATION_EXECUTABLE} PRIVATE
    ${PROJECT_SOURCE_DIR}/client.qrc
    ${PROJECT_SOURCE_DIR}/core_theme.qrc)

generate_theme(${APPLICATION_EXECUTABLE})
set(theme_dir ${OEM_THEME_DIR}/theme)

# add executable icon on windows and osx
if(APPLE)
    file(GLOB_RECURSE OWNCLOUD_SIDEBAR_ICONS "${theme_dir}/colored/*-${APPLICATION_ICON_NAME}-icon-sidebar.png")
    MESSAGE(STATUS "OWNCLOUD_SIDEBAR_ICONS: ${APPLICATION_ICON_NAME}: ${OWNCLOUD_SIDEBAR_ICONS}")
endif()

ecm_add_app_icon(appIcons ICONS "${OWNCLOUD_ICONS}" SIDEBAR_ICONS "${OWNCLOUD_SIDEBAR_ICONS}" OUTFILE_BASENAME "${APPLICATION_ICON_NAME}")
target_sources(${APPLICATION_EXECUTABLE} PRIVATE ${appIcons})

if(NOT APPLE)
    if(WIN32)
        configure_file(
          ${CMAKE_CURRENT_SOURCE_DIR}/version.rc.in
          ${CMAKE_CURRENT_BINARY_DIR}/version.rc
          @ONLY)
        target_sources(${APPLICATION_EXECUTABLE} PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/version.rc)
        if(NOT MSVC)
            target_sources(${APPLICATION_EXECUTABLE} PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/manifest-mingw.rc)
        endif()
    else()
        file(GLOB _icons "${theme_dir}/colored/*-${APPLICATION_ICON_NAME}-icon.png")
        foreach(_file ${_icons})
            string(REPLACE "${theme_dir}/colored/" "" _res ${_file})
            string(REPLACE "-${APPLICATION_ICON_NAME}-icon.png" "" _res ${_res})
            install(FILES ${_file} RENAME ${APPLICATION_ICON_NAME}.png DESTINATION ${DATADIR}/icons/hicolor/${_res}x${_res}/apps)
        endforeach(_file)
    endif()

    install(FILES ${client_I18N} DESTINATION ${SHAREDIR}/${APPLICATION_EXECUTABLE}/i18n)

    # we may not add MACOSX_BUNDLE here, if not building one
else()
    target_sources(${APPLICATION_EXECUTABLE} PRIVATE ${OWNCLOUD_BUNDLED_RESOURCES})

    set_source_files_properties(
      ${client_I18N}
      PROPERTIES
      MACOSX_PACKAGE_LOCATION Resources/Translations
      )

    set_source_files_properties(
      ${OWNCLOUD_BUNDLED_RESOURCES}
      PROPERTIES
      MACOSX_PACKAGE_LOCATION Resources
      )

  set_target_properties(${APPLICATION_EXECUTABLE} PROPERTIES MACOSX_BUNDLE_INFO_PLIST ${CMAKE_CURRENT_SOURCE_DIR}/MacOSXBundleInfo.plist)
endif()

install(TARGETS ${APPLICATION_EXECUTABLE} ${KDE_INSTALL_TARGETS_DEFAULT_ARGS})

if(UNIX AND NOT APPLE)
    include(libcloudproviders/libcloudproviders.cmake)

    configure_file(${CMAKE_SOURCE_DIR}/mirall.desktop.in
                   ${CMAKE_CURRENT_BINARY_DIR}/${APPLICATION_EXECUTABLE}.desktop)
    install(FILES  ${CMAKE_CURRENT_BINARY_DIR}/${APPLICATION_EXECUTABLE}.desktop DESTINATION ${DATADIR}/applications )

    configure_file(owncloud.xml.in ${APPLICATION_EXECUTABLE}.xml)
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${APPLICATION_EXECUTABLE}.xml DESTINATION ${DATADIR}/mime/packages )

    find_package(SharedMimeInfo)
    if(SharedMimeInfo_FOUND)
        update_xdg_mimetypes( ${DATADIR}/mime/packages )
    endif(SharedMimeInfo_FOUND)
endif()
