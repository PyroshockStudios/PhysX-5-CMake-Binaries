cmake_minimum_required(VERSION 3.10)
project(PhysX5CMakeBinaries)

# -------------------------------------------------------------------------
# 1. Setup Paths
# -------------------------------------------------------------------------
# We need to set the binary directory based on the platform FIRST.
if(WIN32)
    set(PX_BIN_DIR "${CMAKE_CURRENT_SOURCE_DIR}/win.x86_64.vc145.md")
elseif(UNIX AND NOT APPLE)
    message(STATUS "Using x86-84 binaries built with the official SteamOS docker...")
    set(PX_BIN_DIR "${CMAKE_CURRENT_SOURCE_DIR}/linux.x86_64.gcc_steamos") 
elseif(APPLE)
    message(FATAL_ERROR "PhysX binaries path not defined for apple!")
    #set(PX_BIN_DIR "${CMAKE_CURRENT_SOURCE_DIR}/mac.x86_64")
else()
    message(FATAL_ERROR "PhysX binaries path not defined for this platform!")
endif()

# -------------------------------------------------------------------------
# 2. Helper Macro
# -------------------------------------------------------------------------
macro(add_physx_module NAME DLL_NAME LIB_NAME)
    add_library(${NAME} SHARED IMPORTED GLOBAL)
    
    if(WIN32)
        # Windows: Needs .lib (Import) and .dll (Runtime)
        set_target_properties(${NAME} PROPERTIES
            IMPORTED_IMPLIB_RELEASE "${PX_BIN_DIR}/release/${LIB_NAME}.lib"
            IMPORTED_IMPLIB_DEBUG   "${PX_BIN_DIR}/debug/${LIB_NAME}.lib"
            IMPORTED_LOCATION_RELEASE "${PX_BIN_DIR}/release/${DLL_NAME}.dll"
            IMPORTED_LOCATION_DEBUG   "${PX_BIN_DIR}/debug/${DLL_NAME}.dll"
        )
    elseif(UNIX AND NOT APPLE)
        # Linux: Needs .so
        set_target_properties(${NAME} PROPERTIES
            IMPORTED_LOCATION_RELEASE "${PX_BIN_DIR}/release/lib${DLL_NAME}.so"
            IMPORTED_LOCATION_DEBUG   "${PX_BIN_DIR}/debug/lib${DLL_NAME}.so"
        )
    elseif(APPLE)
        # Mac: Needs .dylib
        set_target_properties(${NAME} PROPERTIES
            IMPORTED_LOCATION_RELEASE "${PX_BIN_DIR}/release/lib${DLL_NAME}.dylib"
            IMPORTED_LOCATION_DEBUG   "${PX_BIN_DIR}/debug/lib${DLL_NAME}.dylib"
        )
    else()
        message(FATAL_ERROR "Platform not supported in macro!")
    endif() 

    # Standard config mapping
    set_target_properties(${NAME} PROPERTIES
        MAP_IMPORTED_CONFIG_MINSIZEREL Release
        MAP_IMPORTED_CONFIG_RELWITHDEBINFO Release
    )
endmacro()

macro(add_physx_static_module NAME LIB_NAME)
    add_library(${NAME} STATIC IMPORTED GLOBAL)
    
    if(WIN32)
        set_target_properties(${NAME} PROPERTIES
            IMPORTED_LOCATION_RELEASE "${PX_BIN_DIR}/release/${LIB_NAME}.lib"
            IMPORTED_LOCATION_DEBUG   "${PX_BIN_DIR}/debug/${LIB_NAME}.lib"
        )
    elseif(UNIX)
        set_target_properties(${NAME} PROPERTIES
            IMPORTED_LOCATION_RELEASE "${PX_BIN_DIR}/release/lib${LIB_NAME}.a"
            IMPORTED_LOCATION_DEBUG   "${PX_BIN_DIR}/debug/lib${LIB_NAME}.a"
        )
    else()
        message(FATAL_ERROR "Platform not supported in macro!")
    endif() 

    # Standard config mapping
    set_target_properties(${NAME} PROPERTIES
        MAP_IMPORTED_CONFIG_MINSIZEREL Release
        MAP_IMPORTED_CONFIG_RELWITHDEBINFO Release
    )
endmacro()


# -------------------------------------------------------------------------
# 3. Define the Individual Targets
# -------------------------------------------------------------------------

# PhysXFoundation
add_physx_module(PhysXFoundation PhysXFoundation_64 PhysXFoundation_64)

# PhysXCommon
add_physx_module(PhysXCommon PhysXCommon_64 PhysXCommon_64)
target_link_libraries(PhysXCommon INTERFACE PhysXFoundation)

# PhysXCore
add_physx_module(PhysXCore PhysX_64 PhysX_64)
target_link_libraries(PhysXCore INTERFACE PhysXCommon)

# PhysXCooking
add_physx_module(PhysXCooking PhysXCooking_64 PhysXCooking_64)
target_link_libraries(PhysXCooking INTERFACE PhysXCore)

add_physx_static_module(PhysXExtensions PhysXExtensions_static_64)
target_link_libraries(PhysXExtensions INTERFACE PhysXCore)

add_physx_static_module(PhysXVehicle2 PhysXVehicle2_static_64)
target_link_libraries(PhysXVehicle2 INTERFACE PhysXCore)

# -------------------------------------------------------------------------
# 5. Create the Main "PhysX5" Interface Library
# -------------------------------------------------------------------------
add_library(physx5 INTERFACE)

target_include_directories(physx5 INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}/include")

target_link_libraries(physx5 INTERFACE
    PhysXVehicle2 
    PhysXExtensions 
    PhysXCooking 
    PhysXCore
)

add_library(physx5::physx5 ALIAS physx5)
