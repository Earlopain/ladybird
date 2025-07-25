# These are the minimal set of sources needed to build the code generators. We separate them to allow
# LibCore to depend on generated sources.
set(SOURCES
    ArgsParser.cpp
    Directory.cpp
    DirectoryEntry.cpp
    DirIterator.cpp
    Environment.cpp
    File.cpp
    StandardPaths.cpp
    Version.cpp
)

if (WIN32)
    list(APPEND SOURCES
        ProcessWindows.cpp
        SocketpairWindows.cpp
        SystemWindows.cpp)
else()
    list(APPEND SOURCES
        Process.cpp
        System.cpp)
endif()

ladybird_lib(LibCoreMinimal coreminimal)

if (WIN32)
    find_path(DIRENT_INCLUDE_DIR dirent.h REQUIRED)
    target_include_directories(LibCoreMinimal PRIVATE ${DIRENT_INCLUDE_DIR})
endif()

if (${CMAKE_SYSTEM_NAME} MATCHES "NetBSD")
    # NetBSD has its shm_open and shm_unlink functions in librt so we need to link that
    target_link_libraries(LibCoreMinimal PRIVATE rt)
endif()

if (LAGOM_TOOLS_ONLY)
    return()
endif()

set(SOURCES
    ConfigFile.cpp
    DateTime.cpp
    ElapsedTimer.cpp
    Event.cpp
    EventLoop.cpp
    EventLoopImplementation.cpp
    EventReceiver.cpp
    MappedFile.cpp
    MimeData.cpp
    Notifier.cpp
    Resource.cpp
    ResourceImplementation.cpp
    ResourceImplementationFile.cpp
    SystemServerTakeover.cpp
    ThreadEventQueue.cpp
    Timer.cpp
)

if (WIN32)
    # FIXME: Support UDPServer and TCPServer on Windows
    list(APPEND SOURCES
        SocketWindows.cpp
        AnonymousBufferWindows.cpp
        EventLoopImplementationWindows.cpp)
else()
    list(APPEND SOURCES
        Command.cpp
        Socket.cpp
        AnonymousBuffer.cpp
        EventLoopImplementationUnix.cpp
        UDPServer.cpp
        TCPServer.cpp)
endif()

if (NOT WIN32 AND NOT EMSCRIPTEN)
    list(APPEND SOURCES LocalServer.cpp)
endif()

# FIXME: Implement these for other systems.
if (LINUX AND NOT EMSCRIPTEN)
    list(APPEND SOURCES
        FileWatcherLinux.cpp
        Platform/ProcessStatisticsLinux.cpp
        TimeZoneWatcherLinux.cpp
    )
elseif (APPLE AND NOT IOS)
    list(APPEND SOURCES
        FileWatcherMacOS.mm
        Platform/ProcessStatisticsMach.cpp
        TimeZoneWatcherMacOS.mm
    )
else()
    list(APPEND SOURCES
        FileWatcherUnimplemented.cpp
        Platform/ProcessStatisticsUnimplemented.cpp
        TimeZoneWatcherUnimplemented.cpp
    )
endif()

if (APPLE OR CMAKE_SYSTEM_NAME STREQUAL "GNU")
    list(APPEND SOURCES MachPort.cpp)
endif()

if (APPLE)
    list(APPEND SOURCES IOSurface.cpp)
endif()

ladybird_lib(LibCore core)
target_link_libraries(LibCore PRIVATE LibUnicode LibURL Threads::Threads)
target_link_libraries(LibCore PUBLIC LibCoreMinimal)

if (${CMAKE_SYSTEM_NAME} MATCHES "NetBSD")
    # NetBSD has its shm_open and shm_unlink functions in librt so we need to link that
    target_link_libraries(LibCore PRIVATE rt)
endif()
if (${CMAKE_SYSTEM_NAME} MATCHES "SunOS")
    # Solaris has socket and networking related functions in two extra libraries
    target_link_libraries(LibCore PRIVATE nsl socket)
endif()
if (HAIKU)
    # Haiku has networking related functions in the network library
    target_link_libraries(LibCore PRIVATE network)
endif()

if (APPLE)
    target_link_libraries(LibCore PUBLIC "-framework CoreFoundation")
    target_link_libraries(LibCore PUBLIC "-framework CoreServices")
    target_link_libraries(LibCore PUBLIC "-framework Foundation")
    target_link_libraries(LibCore PUBLIC "-framework IOSurface")
endif()

if (ANDROID)
    target_link_libraries(LibCore PRIVATE log)
endif()

if (ENABLE_SWIFT)
    set(SWIFT_EXCLUDE_HEADERS "SocketAddressWindows.h")
    if(WIN32)
        list(APPEND SWIFT_EXCLUDE_HEADERS "EventLoopImplementationUnix.h")
    else()
        list(APPEND SWIFT_EXCLUDE_HEADERS "EventLoopImplementationWindows.h")
    endif()
    if (NOT APPLE)
        list(APPEND SWIFT_EXCLUDE_HEADERS
            IOSurface.h
            MachPort.h
            MachMessageTypes.h
            ProcessStatisticsMach.h
        )
    endif()

    generate_clang_module_map(LibCore EXCLUDE_FILES ${SWIFT_EXCLUDE_HEADERS})
    target_sources(LibCore PRIVATE
        EventSwift.mm
        EventLoopExecutor.swift)
    set_source_files_properties(EventSwift.mm PRIVATE PROPERTIES COMPILE_FLAGS -fblocks)
    target_link_libraries(LibCore PRIVATE AK)
    add_swift_target_properties(LibCore LAGOM_LIBRARIES AK)
endif()
