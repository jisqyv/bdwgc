#
# Copyright (c) 1994 by Xerox Corporation.  All rights reserved.
# Copyright (c) 1996 by Silicon Graphics.  All rights reserved.
# Copyright (c) 1998 by Fergus Henderson.  All rights reserved.
# Copyright (c) 2000-2010 by Hewlett-Packard Company.  All rights reserved.
##
# THIS MATERIAL IS PROVIDED AS IS, WITH ABSOLUTELY NO WARRANTY EXPRESSED
# OR IMPLIED.  ANY USE IS AT YOUR OWN RISK.
##
# Permission is hereby granted to use or copy this program
# for any purpose,  provided the above notices are retained on all copies.
# Permission to modify the code and to distribute modified code is granted,
# provided the above notices are retained, and a notice that the code was
# modified is included with the above copyright notice.
##

#
#  get cmake and run:
#    cmake -G "Visual Studio 8 2005"
#  in the same dir as this file
#  this will generate gc.sln
#

SET(CMAKE_LEGACY_CYGWIN_WIN32 0) # Remove when CMake >= 2.8.4 is required

PROJECT(gc)

CMAKE_MINIMUM_REQUIRED(VERSION 2.6)

ADD_DEFINITIONS("-D_CRT_SECURE_NO_DEPRECATE
                 -DALL_INTERIOR_POINTERS
                 -DGC_BUILD
")

IF(APPLE)
    SET(CMAKE_OSX_ARCHITECTURES "ppc;i386;x86_64" CACHE STRING "Build architectures for Mac OS X" FORCE)
ENDIF(APPLE)

#LIBATOMIC #TODO
#ADD_LIBRARY(atomic_ops STATIC )
#SET_TARGET_PROPERTIES(atomic_ops PROPERTIES COMPILE_FLAGS -DNO_DEBUGGING)


#LIBGC

INCLUDE_DIRECTORIES(include)
INCLUDE_DIRECTORIES(libatomic_ops/src)

SET(SRC alloc.c reclaim.c allchblk.c misc.c mach_dep.c os_dep.c
        mark_rts.c headers.c mark.c obj_map.c blacklst.c finalize.c
        new_hblk.c dbg_mlc.c malloc.c stubborn.c dyn_load.c
        typd_mlc.c ptr_chck.c gc_cpp.cc mallocx.c checksums.c
        thread_local_alloc.c)
SET(LIBS)
OPTION(enable_threads "TODO" NO)
IF(enable_threads)
        FIND_PACKAGE(Threads REQUIRED)
        MESSAGE("Thread Model: ${CMAKE_THREAD_LIBS_INIT}" )
        INCLUDE_DIRECTORIES(${Threads_INCLUDE_DIR})
        SET(LIBS ${LIBS} ${Threads_LIBRARIES})
ENDIF(enable_threads)

OPTION(enable_parallel_mark "Parallelize marking and free list construction" NO)

#IF(Threads_FOUND)
#       ADD_DEFINITIONS("")
#ELSE
#       MESSAGE("Parallel mark requires enable_threads ON" )
#ENDIF(Threads_FOUND)

IF(enable_parallel_mark)
ENDIF(enable_parallel_mark)

OPTION(enable_cplusplus "install C++ support" NO)

SET(_HOST ${CMAKE_HOST_SYSTEM_PROCESSOR}--${CMAKE_SYSTEM}) #FIXME missing the vendor field.Use lowercase

STRING(TOLOWER ${_HOST} HOST)
MESSAGE("HOST = ${HOST}")

#Thread Detection. Relying on cmake for lib an includes.
#TODO check cmake detection
IF(CMAKE_USE_PTHREADS_INIT)
        SET(SRC ${SRC} pthread_start.c pthread_support.c pthread_stop_world.c)
        IF( "HOST" MATCHES x86-.*-linux.*|ia64-.*-linux.*|i586-.*-linux.*|i686-.*-linux.*|x86_64-.*-linux.*|alpha-.*-linux.*|sparc.*-.*-linux.*)
                ADD_DEFINITIONS("-DGC_LINUX_THREADS")
                ADD_DEFINITIONS("-D_REENTRANT")
                IF (${enable_parallel_mark})
                        ADD_DEFINITIONS("-DPARALLEL_MARK")
                ENDIF()
                ADD_DEFINITIONS("-DTHREAD_LOCAL_ALLOC")
                MESSAGE("Explicit GC_INIT() calls may be required.")
        ENDIF()
        IF ( "HOST" MATCHES .*-.*-linux.*)
                ADD_DEFINITIONS("-DGC_LINUX_THREADS")
                ADD_DEFINITIONS("-D_REENTRANT")
        ENDIF()
        IF ( "HOST" MATCHES .*-.*-aix.*)
                ADD_DEFINITIONS("-DGC_AIX_THREADS")
                ADD_DEFINITIONS("-D_REENTRANT")
        ENDIF()
        IF ( "HOST" MATCHES .*-.*-hpux11.*)
                MESSAGE("Only HP/UX 11 POSIX threads are supported.")
                ADD_DEFINITIONS("-DGC_HPUX_THREADS")
                ADD_DEFINITIONS("-D_POSIX_C_SOURCE=199506L") #TODO test -DVAR=value. Alternative is COMPILE_DEFINITIONS property
                IF (${enable_parallel_mark})
                        ADD_DEFINITIONS("-DPARALLEL_MARK")
                ENDIF()
                MESSAGE("Explicit GC_INIT() calls may be required.")
                ADD_DEFINITIONS("-D_REENTRANT") #TODO
        ENDIF()
        IF ( "HOST" MATCHES .*-.*-hpux10.*)
                MESSAGE("Only HP/UX 11 POSIX threads are supported.")
        ENDIF()
        IF ( "HOST" MATCHES .*-.*-openbsd.*)
                ADD_DEFINITIONS("-DGC_OPENBSD_THREADS")
        ENDIF()
        IF ( "HOST" MATCHES .*-.*-freebsd.*)
                MESSAGE("FreeBSD does not yet fully support threads with Boehm GC.")
                ADD_DEFINITIONS("-DGC_FREEBSD_THREADS")
        ENDIF()
        IF ( "HOST" MATCHES .*-.*-kfreebsd.*-gnu)
                ADD_DEFINITIONS("-DGC_FREEBSD_THREADS")
                ADD_DEFINITIONS("-D_REENTRANT")
                IF (${enable_parallel_mark})
                        ADD_DEFINITIONS("-DPARALLEL_MARK")
                ENDIF()
                ADD_DEFINITIONS(THREAD_LOCAL_ALLOC")
                ADD_DEFINITIONS(USE_COMPILER_TLS")
        ENDIF()
        IF ( "HOST" MATCHES .*-.*-gnu.*)
                ADD_DEFINITIONS("-DGC_GNU_THREADS")
                ADD_DEFINITIONS("-D_REENTRANT")
                ADD_DEFINITIONS("-DTHREAD_LOCAL_ALLOC")
        ENDIF()
        IF ( "HOST" MATCHES .*-.*-netbsd.*)
                MESSAGE("Only on NetBSD 2.0 or later.")
                ADD_DEFINITIONS("-DGC_NETBSD_THREADS")
                ADD_DEFINITIONS("-D_REENTRANT")
                ADD_DEFINITIONS("-D_PTHREADS")
        ENDIF()
        IF ( "HOST" MATCHES .*-.*-solaris.*)
                ADD_DEFINITIONS("-DGC_SOLARIS_THREADS")
                ADD_DEFINITIONS("-DTHREAD_LOCAL_ALLOC")
#TODO
#       if test "$GCC" != yes; then
#          CFLAGS="$CFLAGS -O"
#          need_atomic_ops_asm=true
#        fi

        ENDIF()
        IF ( "HOST" MATCHES .*-.*-irix.*)
                ADD_DEFINITIONS("-DGC_IRIX_THREADS")
        ENDIF()
        IF ( "HOST" MATCHES .*-.*-cygwin.*)
                ADD_DEFINITIONS("-DGC_THREADS")
                IF (${enable_parallel_mark})
                        ADD_DEFINITIONS("-DPARALLEL_MARK")
                ENDIF()
                ADD_DEFINITIONS("-DTHREAD_LOCAL_ALLOC")

#TODO
#       win32_threads=true
        ENDIF()
        IF ( "HOST" MATCHES .*-.*-darwin.*)
                ADD_DEFINITIONS("-DGC_DARWIN_THREADS")
                ADD_DEFINITIONS("-DTHREAD_LOCAL_ALLOC")
                MESSAGE("Explicit GC_INIT() calls may be required.")
                SET(SRC ${SRC} darwin_stop_world.c)
                IF (${enable_parallel_mark})
                       ADD_DEFINITIONS("-DPARALLEL_MARK")
                ENDIF()
                #TODO
                #darwin_threads=true
        ENDIF()
        IF ( "HOST" MATCHES .*-.*-osf*)
                ADD_DEFINITIONS("-DGC_OSF1_THREADS")
                IF (${enable_parallel_mark})
                        ADD_DEFINITIONS("-DPARALLEL_MARK")
                        ADD_DEFINITIONS("-DTHREAD_LOCAL_ALLOC")
                        MESSAGE("Explicit GC_INIT() calls may be required.")
                        # May want to enable it in other cases, too.
                        # Measurements haven't yet been done.
                ENDIF()
        ENDIF()
        IF ( "HOST" MATCHES .*-.*-linux.*)
                ADD_DEFINITIONS("-DGC_LINUX_THREADS")
                ADD_DEFINITIONS("-D_REENTRANT")
        ENDIF()
ENDIF(CMAKE_USE_PTHREADS_INIT)

IF(CMAKE_USE_WIN32_THREADS_INIT)
        ADD_DEFINITIONS("-DGC_THREADS")
        #win32_threads=true   TODO
        IF (${enable_parallel_mark})
                ADD_DEFINITIONS("-DPARALLEL_MARK")
                ADD_DEFINITIONS("-DTHREAD_LOCAL_ALLOC")
        ENDIF()
        ADD_DEFINITIONS("-DEMPTY_GETENV_RESULTS") #TODO test
        SET(SRC ${SRC} win32_threads.c)
ENDIF(CMAKE_USE_WIN32_THREADS_INIT)

OPTION(enable_gcj_support "Support for gcj" NO)
IF(enable_gcj_support)
        ADD_DEFINITIONS("-DGC_GCJ_SUPPORT")
ENDIF(enable_gcj_support)


ADD_LIBRARY(          gc-lib   STATIC      ${SRC})
SET_TARGET_PROPERTIES(gc-lib   PROPERTIES
                      COMPILE_DEFINITIONS GC_NOT_DLL)
#TODO TARGET_LINK_LIBRARIES(...  ...  ${LIBS})

ADD_LIBRARY(          gcmt-lib STATIC      ${SRC})
SET_TARGET_PROPERTIES(gcmt-lib PROPERTIES
                      COMPILE_DEFINITIONS GC_NOT_DLL)

ADD_LIBRARY(          gcmt-dll  SHARED     ${SRC})

IF(WIN32)
  ADD_EXECUTABLE(cord cord/cordbscs.c cord/cordxtra.c
                 cord/tests/de.c cord/tests/de_win.c)
  SET_TARGET_PROPERTIES(cord PROPERTIES WIN32_EXECUTABLE TRUE)
  SET_TARGET_PROPERTIES(cord    PROPERTIES
                      COMPILE_DEFINITIONS GC_NOT_DLL)
  TARGET_LINK_LIBRARIES(cord gc-lib)
  TARGET_LINK_LIBRARIES(cord gdi32)
ENDIF(WIN32)

ADD_SUBDIRECTORY(tests)
