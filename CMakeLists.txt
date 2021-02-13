#[[
   Copyright (c) 2021 Sibi Siddharthan

   Distributed under the MIT license.
   Refer to the LICENSE file at the root directory of the parent project
   for details.
]]

cmake_minimum_required(VERSION 3.12)
project(pinentry
		VERSION 1.1.1
		LANGUAGES C)

add_compile_definitions(HAVE_CONFIG_H)

set(PACKAGE \"${PROJECT_NAME}\")
set(PACKAGE_BUGREPORT \"https://bugs.gnupg.org\")
set(PACKAGE_NAME \"${PROJECT_NAME}\")
set(PACKAGE_STRING \"${PROJECT_NAME}\ ${PROJECT_VERSION}\")
set(PACKAGE_TARNAME \"${PROJECT_NAME}\")
set(PACKAGE_URL \"\")
set(PACKAGE_VERSION \"${PROJECT_VERSION}\")
set(VERSION \"${PROJECT_VERSION}\")

#find_package(GPGRT REQUIRED)
#find_package(ASSUAN REQUIRED)
find_package(Iconv)
include(UtilityFunctions)
include(CheckCSourceCompiles)
include(CheckCSourceRuns)

if(Iconv_FOUND)
	set(HAVE_ICONV 1)
	set(ICONV_CONST " ")
endif()

if(WIN32)
	set(HAVE_W32_SYSTEM 1)
	set(HAVE_W64_SYSTEM 1)
	set(HAVE_DOSISH_SYSTEM 1)
	set(HAVE_DRIVE_LETTERS 1)
endif()

if(UNIX)
	set(CURSES_NEED_WIDE 1)
	find_package(Curses)
	if(CURSES_FOUND)
		set(PINENTRY_CURSES 1)
	else()
		set(PINENTRY_TTY 1)
	endif()
endif()

check_c_headers(inttypes.h langinfo.h locale.h memory.h stdint.h stdlib.h strings.h string.h
sys/mman.h sys/stat.h sys/types.h termio.h unistd.h utime.h wchar.h)

check_functions(mmap plock seteuid stpcpy mlock)

foreach(t byte ulong)
string(TOUPPER ${t} ut)
	check_c_source_compiles("
		#define _GNU_SOURCE 1
		#include <stdlib.h>
		#include <sys/types.h>
		int
		main ()
		{
			#undef ${t}
			int a = sizeof(${t});
			return 0;
		}"
	HAVE_${ut}_TYPEDEF)
endforeach()


if(HAVE_MLOCK)
check_c_source_runs("
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <sys/mman.h>
#include <sys/types.h>
#include <fcntl.h>

int main()
{
    char *pool;
    int err;
    long int pgsize;

#if ${HAVE_SYSCONF} && defined(_SC_PAGESIZE)
    pgsize = sysconf (_SC_PAGESIZE);
#elif  ${HAVE_GETPAGESIZE}
    pgsize = getpagesize();
#else
    pgsize = -1;
#endif

    if (pgsize == -1)
      pgsize = 4096;

    pool = malloc( 4096 + pgsize );
    if( !pool )
        return 2;
    pool += (pgsize - ((long int)pool % pgsize));

    err = mlock( pool, 4096 );
    if( !err || errno == EPERM || errno == EAGAIN)
        return 0; /* okay */

    return 1;  /* hmmm */
}"
MLOCK_OKAY)
if(NOT MLOCK_OKAY)
	set(HAVE_BROKEN_MLOCK 1)
endif()
endif()

configure_file(pinentry-config.h.in config.h)

include_directories(${CMAKE_CURRENT_BINARY_DIR} src secmem pinentry)

set(pinentry_SOURCES
pinentry.h pinentry.c argparse.c argparse.h
password-cache.h password-cache.c)

list(TRANSFORM pinentry_SOURCES PREPEND "pinentry/")

add_library(pinentry OBJECT ${pinentry_SOURCES})
target_link_libraries(pinentry GPGRT::GPGRT ASSUAN::ASSUAN)

set(secmem_SOURCES
memory.h
secmem-util.h
util.h
secmem.c
util.c
secmem++.h)

list(TRANSFORM secmem_SOURCES PREPEND "secmem/")
add_library(secmem OBJECT ${secmem_SOURCES})

if(WIN32)
	set(pinentry_exe pinentry-w32)
	add_executable(${pinentry_exe} w32/main.c w32/pinentry-w32.rc)
else()
	if(PINENTRY_CURSES)
		set(pinentry_exe pinentry-curses)
		add_executable(${pinentry_exe} curses/pinentry-curses.c pinentry/pinentry-curses.c)
		target_include_directories(${pinentry_exe} PRIVATE ${CURSES_INCLUDE_DIRS})
		target_link_libraries(${pinentry_exe} ${CURSES_LIBRARIES})
	else() # PINENTRY_TTY
		set(pinentry_exe pinentry-tty)
		add_executable(${pinentry_exe} tty/pinentry-tty.c)
	endif()
endif()
	
set_target_properties(${pinentry_exe} PROPERTIES RUNTIME_OUTPUT_NAME "pinentry")
target_link_libraries(${pinentry_exe} pinentry secmem)
install(TARGETS ${pinentry_exe} RUNTIME DESTINATION bin)
#install(FILES doc/pinentry.info DESTINATION share/info)
