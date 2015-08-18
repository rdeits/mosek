cmake_minimum_required(VERSION 2.8.0)

project(mosek)

# pull in the pods macros. See cmake/pods.cmake for documentation
set(POD_NAME mosek-pod)
include(cmake/pods.cmake)
include(cmake/matlab_pods.cmake)
include(cmake/mex.cmake)

# Set up the system name and various paths
if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
	set(PLATFORM_NAME "osx64x86" )
elseif (${CMAKE_SYSTEM_NAME} MATCHES "Linux")
	execute_process(COMMAND uname -m
		            OUTPUT_VARIABLE UNAME_M)
	if (${UNAME_M} MATCHES "x86_64")
		set(PLATFORM_NAME "linux64x86" )
	elseif (${UNAME_M} MATCHES "x86")
		set(PLATFORM_NAME "linux32x86" )
	else()
		message(FATAL_ERROR "Unrecognized system variant: ${CMAKE_SYSTEM_NAME} ${UNAME_M}")
	endif()
elseif (${CMAKE_SYSTEM_NAME} MATCHES "Windows")
	message(FATAL_ERROR "Sorry, I haven't yet been able to support Windows. Please open an issue on the github site if this is important for you.")
else()
	message(FATAL_ERROR "Unrecognized operating system name: ${CMAKE_SYSTEM_NAME}")
endif()

set(DL_PATH "http://download.mosek.com/stable/7")
set(DL_NAME "mosektools${PLATFORM_NAME}.tar.bz2")
set(MOSEK_PLATFORM_PATH "${CMAKE_CURRENT_BINARY_DIR}/mosek/7/tools/platform/${PLATFORM_NAME}")

# Check for the mosek license file
if (NOT EXISTS $ENV{HOME}/mosek/mosek.lic)
    message(FATAL_ERROR "You do not appear to have a license for mosek installed in $ENV{HOME}/mosek/mosek.lic\n If you are an academic user, you can open the following url in your favorite browser and request the license:\n     http://license.mosek.com/license2/academic/\n Then check your email for the license file and put it in $ENV{HOME}/mosek/mosek.lic\n Otherwise visit https://mosek.com/resources/trial-license to request a trial license.")
endif()

# Download and unzip the source
if (NOT EXISTS ${MOSEK_PLATFORM_PATH}/h/mosek.h)
	message("Downloading and extracting Mosek files from ${DL_PATH}/${DL_NAME} ...")
	file(DOWNLOAD "${DL_PATH}/${DL_NAME}" "${CMAKE_CURRENT_BINARY_DIR}/${DL_NAME}" SHOW_PROGRESS)
	execute_process(
	     COMMAND ${CMAKE_COMMAND} -E tar xzf "${CMAKE_CURRENT_BINARY_DIR}/${DL_NAME}"
	     WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
	     )
	message("...done")
endif()

# Install support for various languages:

############# C / C++ ###############
pods_install_headers(${MOSEK_PLATFORM_PATH}/h/mosek.h DESTINATION mosek)
install(DIRECTORY ${MOSEK_PLATFORM_PATH}/bin/ DESTINATION lib
        FILES_MATCHING PATTERN "*mosek64*")
install(DIRECTORY ${MOSEK_PLATFORM_PATH}/bin/ DESTINATION lib
        FILES_MATCHING PATTERN "*iomp5*")
pods_install_pkg_config_file(mosek
  LIBS -lmosek64 -L${CMAKE_INSTALL_PREFIX}/lib
  CFLAGS -I${CMAKE_INSTALL_PREFIX}/include
  REQUIRES
  VERSION 7.0.0)

############# Python ################
pods_install_python_packages("${MOSEK_PLATFORM_PATH}/python/2/")

############# Matlab ################
if (MATLAB_FOUND)
	pods_configure_matlab_paths()
	set(mosek_addpathfile ${CMAKE_CURRENT_BINARY_DIR}/matlab/addpath_mosek.m)
	execute_process(COMMAND matlab -r "info = ver('matlab'); fprintf(1, '%s', info.Version);"
	                OUTPUT_VARIABLE MATLAB_VERSION)
	if (MATLAB_VERSION VERSION_GREATER 8.0)
		set(MOSEK_MATLAB_DIR r2013a)
	elseif(MATLAB_VERSION VERSION_GREATER 7.13)
		set(MOSEK_MATLAB_DIR r2012a)
	else()
		message(FATAL_ERROR "Mosek requires MATLAB 7.14 (R2012a) or higher")
	endif()

	file(WRITE ${mosek_addpathfile}
	    "function addpath_mosek()\n"
	    "  mfiledir = fullfile('${CMAKE_CURRENT_BINARY_DIR}', 'mosek', '7', 'toolbox', '${MOSEK_MATLAB_DIR}');\n"
	    "  addpath(mfiledir);\n"
	    "  jarfiledir = fullfile('${CMAKE_INSTALL_PREFIX}', 'share', 'java');\n"
        "  if exist(jarfiledir, 'dir') \n"
        "    javaaddpathIfNew(jarfiledir);\n"
        "    for jarfile = dir(fullfile(jarfiledir, '*.jar'))'; \n"
        "      javaaddpathIfNew(fullfile(jarfiledir, jarfile.name)); \n"
        "    end\n"
        "  end\n
        "end\n"
        "\n"
        "function javaaddpathIfNew(p)\n"
        "  % Add a .jar to the dynamic java classpath only if it hasn't already been added\n"
        "  if ~any(cellfun(@(x) strcmp(x, p), javaclasspath('-dynamic')))\n"
        "    javaaddpath(p);\n"
        "  end\n"
        "end\n"
        )

	set(mosek_rmpathfile ${CMAKE_CURRENT_BINARY_DIR}/matlab/rmpath_mosek.m)
	file(WRITE ${mosek_rmpathfile}
	    "function rmpath_mosek()\n"
	    "  mfiledir = fullfile('${CMAKE_CURRENT_BINARY_DIR}', 'mosek', '7', 'toolbox', '${MOSEK_MATLAB_DIR}');\n"
	    "  rmpath(mfiledir);\n"
	    "end\n"
	)
	install(FILES ${mosek_addpathfile} ${mosek_rmpathfile} DESTINATION matlab)
endif()

############ Java and Matlab ################
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/mosek/7/tools/platform/${PLATFORM_NAME}/bin/mosekmatlab.jar DESTINATION share/java)

