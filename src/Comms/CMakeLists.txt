add_subdirectory(MockLink)

find_package(Qt6 REQUIRED COMPONENTS Core Network Qml Test Widgets)

qt_add_library(Comms STATIC
    LinkConfiguration.cc
    LinkConfiguration.h
    LinkInterface.cc
    LinkInterface.h
    LinkManager.cc
    LinkManager.h
    LogReplayLink.cc
    LogReplayLink.h
    MAVLinkProtocol.cc
    MAVLinkProtocol.h
    TCPLink.cc
    TCPLink.h
    UdpIODevice.cc
    UdpIODevice.h
    UDPLink.cc
    UDPLink.h

#    # Add the XPC source file
#    ${CMAKE_SOURCE_DIR}/src/XPlaneConnect/xplaneConnect.c
#    ${CMAKE_SOURCE_DIR}/src/XPlaneConnect/xplaneConnect.h

    xplaneConnect.c
    xplaneConnect.h

)

target_link_libraries(Comms
    PRIVATE
        Qt6::Qml
        Qt6::Test
        AirLink
        MockLink
        Settings
        Vehicle
    PUBLIC
        Qt6::Core
        Qt6::Network
        MAVLink
        QGC
        QmlControls
        Utilities
)

target_include_directories(Comms PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})

############# Serial

if(QGC_NO_SERIAL_LINK)
    target_compile_definitions(Comms PUBLIC NO_SERIAL_LINK)
else()
    target_sources(Comms
        PRIVATE
            QGCSerialPortInfo.cc
            QGCSerialPortInfo.h
            SerialLink.cc
            SerialLink.h
    )
    target_link_libraries(Comms
        PRIVATE
            GPS
            PositionManager
    )
    if(ANDROID)
        add_subdirectory(${CMAKE_SOURCE_DIR}/android/libs/qtandroidserialport qtandroidserialport.build)
        target_sources(qtandroidserialport
            PRIVATE
                ${CMAKE_SOURCE_DIR}/android/src/AndroidSerial.cc
                ${CMAKE_SOURCE_DIR}/android/src/AndroidSerial.h
        )
        target_include_directories(qtandroidserialport PUBLIC ${CMAKE_SOURCE_DIR}/android/src)
        target_link_libraries(Comms PUBLIC qtandroidserialport)
    else()
        find_package(Qt6 REQUIRED COMPONENTS SerialPort)
        target_link_libraries(Comms PUBLIC Qt6::SerialPort)
    endif()
endif()

############# Bluetooth

option(QGC_ENABLE_BLUETOOTH "Enable Bluetooth Links" ON)
if(QGC_ENABLE_BLUETOOTH)
    find_package(Qt6 REQUIRED COMPONENTS Bluetooth)
    target_link_libraries(Comms PUBLIC Qt6::Bluetooth)
    target_sources(Comms
        PRIVATE
            BluetoothLink.cc
            BluetoothLink.h
    )
    target_compile_definitions(Comms PUBLIC QGC_ENABLE_BLUETOOTH)
endif()

############# ZEROConf

option(QGC_ZEROCONF_ENABLED "Enable ZeroConf Compatibility" OFF)
if(QGC_ZEROCONF_ENABLED)
    set(BUILD_SHARED_LIBS OFF CACHE INTERNAL "")
    set(BUILD_DOC OFF CACHE INTERNAL "")
    set(BUILD_EXAMPLES OFF CACHE INTERNAL "")
    set(BUILD_TESTS OFF CACHE INTERNAL "")

    include(FetchContent)
    FetchContent_Declare(qmdnsengine
        GIT_REPOSITORY https://github.com/nitroshare/qmdnsengine.git
        GIT_TAG 4e54bc86c8ed2d4fa2e7449d4ba6a6a2742d9eb1
    )
    FetchContent_MakeAvailable(qmdnsengine)

    target_link_libraries(Comms PRIVATE qmdnsengine)
    target_compile_definitions(Comms PUBLIC QGC_ZEROCONF_ENABLED)
endif()

############# XPlaneConnect

cmake_minimum_required(VERSION 2.8.4)

set(LIBXPLANECONNECT_SRC xplaneConnect.c)

add_library(xplaneconnect_dynamic SHARED ${LIBXPLANECONNECT_SRC})
add_library(xplaneconnect_static  STATIC ${LIBXPLANECONNECT_SRC})

set_target_properties(xplaneconnect_dynamic PROPERTIES OUTPUT_NAME "xplaneconnect")
set_target_properties(xplaneconnect_dynamic PROPERTIES VERSION 1.2 SOVERSION 1)

set_target_properties(xplaneconnect_static  PROPERTIES OUTPUT_NAME "xplaneconnect")

set(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)

install(TARGETS xplaneconnect_dynamic xplaneconnect_static
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib)

install(FILES xplaneConnect.h DESTINATION include/xplaneConnect)

