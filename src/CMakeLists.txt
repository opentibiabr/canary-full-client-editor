
# *****************************************************************************
# Project canary-full-client-editor
# *****************************************************************************
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    project(canary-full-client-editor-debug LANGUAGES CXX)
else()
    project(canary-full-client-editor LANGUAGES CXX)
endif()

find_package(Qt6 REQUIRED COMPONENTS Core Gui Widgets)
find_package(fmt CONFIG REQUIRED)
find_package(Threads REQUIRED)
find_package(ZLIB REQUIRED)
find_package(liblzma CONFIG REQUIRED)
find_package(magic_enum CONFIG REQUIRED)
find_package(spdlog REQUIRED)

set(CMAKE_AUTOMOC ON)  # Enable AUTOMOC to automatically process Q_OBJECT macros
set(CMAKE_AUTOUIC ON)  # Enable Qt Designer .ui file support
set(CMAKE_AUTORCC ON)  # Enable Qt resource file (.qrc) support

# *****************************************************************************
# Options
# *****************************************************************************
option(TOGGLE_BIN_FOLDER "Use build/bin folder for generate compilation files" OFF)
option(OPTIONS_ENABLE_OPENMP "Enable Open Multi-Processing support." ON)
option(DEBUG_LOG "Enable Debug Log" OFF)
option(SPEED_UP_BUILD_UNITY "Compile using build unity for speed up build" ON)

# Build static libs
if(BUILD_STATIC_LIBRARY)
	log_option_enabled("STATIC_LIBRARY")

	if(UNIX AND NOT APPLE)
		set(CMAKE_FIND_LIBRARY_SUFFIXES ".a")
	elseif(APPLE)
		set(CMAKE_FIND_LIBRARY_SUFFIXES ".a" ".dylib")
	endif()
else()
	log_option_disabled("STATIC_LIBRARY")
endif()

# === DEBUG LOG ===
# cmake -DDEBUG_LOG=ON ..
if(DEBUG_LOG)
	add_definitions(-DDEBUG_LOG=ON)
	log_option_enabled("DEBUG LOG")
else()
	log_option_disabled("DEBUG LOG")
endif(DEBUG_LOG)

if (MSVC)
	add_executable(${PROJECT_NAME} "" ../cmake/canary-full-client-editor.rc)

	if(BUILD_STATIC_LIBRARY)
		set_property(TARGET ${PROJECT_NAME} PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
	endif()

	target_compile_options(${PROJECT_NAME} PUBLIC /MP /FS /Zf /EHsc )
else()
	add_executable(${PROJECT_NAME} "")
endif()

# === OpenMP ===
if(OPTIONS_ENABLE_OPENMP)
	log_option_enabled("openmp")
	find_package(OpenMP)
	if(OpenMP_CXX_FOUND)
		target_link_libraries(${PROJECT_NAME} PUBLIC OpenMP::OpenMP_CXX)
	endif()
else()
	log_option_disabled("openmp")
endif()

# === IPO ===
check_ipo_supported(RESULT result OUTPUT output)
if(result)
	set_property(TARGET ${PROJECT_NAME} PROPERTY INTERPROCEDURAL_OPTIMIZATION TRUE)
else()
	message(WARNING "IPO is not supported: ${output}")
endif()


# === UNITY BUILD (compile time reducer) ===
if(SPEED_UP_BUILD_UNITY)
	set_target_properties(${PROJECT_NAME} PROPERTIES UNITY_BUILD ON)
	log_option_enabled("Build unity for speed up compilation")
else()
	# === PRECOMPILED HEADER ===
	target_precompile_headers(${PROJECT_NAME} PRIVATE main.h)
endif()


target_sources(${PROJECT_NAME}
	PRIVATE
	windows/main/mainwindow.cpp
)

target_include_directories(${PROJECT_NAME}
	PRIVATE
	${CMAKE_SOURCE_DIR}/src
	${ZLIB_INCLUDE_DIR}
)

target_link_libraries(${PROJECT_NAME}
	PRIVATE
	${ZLIB_LIBRARIES}
	Qt6::Core Qt6::Gui Qt6::Widgets
	fmt::fmt
	liblzma::liblzma
	spdlog::spdlog
	protobuf
)

## Link compilation files to build/bin folder, else link to the main dir
if (TOGGLE_BIN_FOLDER)
	set_target_properties(${PROJECT_NAME}
			PROPERTIES
			RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
	)
else()
	set_target_properties(${PROJECT_NAME}
			PROPERTIES
			RUNTIME_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/"
	)
endif()
