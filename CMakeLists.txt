##############################################################################
# Simple cmake script to build SQLite module with encryption (SQLCipher)
# for Qt4 with Microsoft Visual Studio.
#
# Encryption is provided by SQLCipher.
# Tested with SQLCipher-2.0.6, sqlite-3.7.14.1, Qt-4.8.3, OpenSSL-1.0.1b
#
# Inspired by
#   https://groups.google.com/d/msg/sqlcipher/ISwSo2Vq-B8/x205cECTiI0J
#   http://www.wiki.crossplatform.ru/index.php/%D0%9F%D1%80%D0%B8%D0%B2%D1%8F%D0%B7%D0%BA%D0%B0_SQLCipher_%D0%BA_Qt
#
# 2012, Sergey Kozlov
#
#
# === How to build
#
# You'll need the following files:
#   * sqlcipher sources (http://sqlcipher.net)
#   * sqlite amalgamation sources (http://sqlite.org)
#   * OpenSSL http://slproweb.com/products/Win32OpenSSL.html
#   * Qt 4.8.3 distribution
#
# Modify the following paths in the Configuration section bellow:
#   SQLITE_SRC_DIR, CIPHER_ROOT, OPENSSL_ROOT
# If you don't want to use system Qt4 then set QT_QMAKE_EXECUTABLE.
# By setting OPENSSL_MT and OPENSSL_STATIC cmake options you may switch
# dynamic runtime libs and static runtime libs.
# You also can modify default definitions in the CIPHER_DEF if you want.
#
# Example cmake commands:
#   cmake -G"NMake Makefiles" -DCMAKE_BUILD_TYPE=Release
#   cmake -G"NMake Makefiles" -DCMAKE_BUILD_TYPE=Debug
#   cmake -G"NMake Makefiles" -DCMAKE_BUILD_TYPE=Release -DOPENSSL_MT=OFF -DOPENSSL_STATIC=ON
# then you run nmake and nmake install.
# After compilation you will have: sqlcipher.lib, qsqlcipher.dll, sqlite.exe
#
#
# === What happens while configuration/compilation
#
# 1) Copy sqlite3.c -> sqlite.c and append #include "crypto.c"
# 2) Copy crypto.h and crypto.c to the single crypto.c
# 3) Append crypto parts from the pager.c to the crypto.c
# 4) Append crypto_impl.c to the crypto.c
# 5) Build sqlcipher.lib using sqlite.c
# 6) Build qsqlcipher.dll using sqlcipher.lib, smain.cpp and Qt's qsql_sqlite.cpp
# 7) Buils sqlite.exe using shell.c and sqlcipher.lib
#
# Enjoy.
##############################################################################

cmake_minimum_required(VERSION 2.8.8)

##########################################################
## Configuration
##########################################################

### SQLite source path
set(SQLITE_SRC_DIR "D:/work/software/sqlcipher/sqlite-amalgamation")

### SQLite Cipher sources root
set(CIPHER_ROOT "D:/work/software/sqlcipher/sqlcipher")

### Definitions
set(CIPHER_DEF "-DSQLITE_HAS_CODEC -DSQLITE_ENABLE_COLUMN_METADATA -DSQLITE_TEMP_STORE=2")

### Win32 OpenSSL instalation path
set(OPENSSL_ROOT "D:/tools/OpenSSL-Win64")

# Uncomment if you want to specify Qt4
set(QT_QMAKE_EXECUTABLE "D:/tools/qt/Qt5.1.1/5.1.1/msvc2012_64/bin/qmake")
set(CMAKE_PREFIX_PATH "D:/tools/qt/Qt5.1.1/5.1.1/msvc2012_64")

option(OPENSSL_MT "Use OpenSSL MT libs instead of MD" OFF)
option(OPENSSL_STATIC "Use OpenSSL static libs instead of dynamic" OFF)
option(BUILD_QT_PLUGIN "Build Qt sql plugin" ON)
option(BUILD_SHELL "Build sqlite shell" ON)

##########################################################
## Tools
##########################################################

### Helper macro to search files
macro(aux_find_file Var Name Path)
    set(${Var} ${Var}-NOTFOUND)
    find_file(${Var} ${Name} PATHS ${Path})
    if (${Var}-NOTFOUND)
        message(FATAL_ERROR "Can't find ${Name}")
    endif()
    message(STATUS "[${Name}] ${${Var}}")
endmacro()

### Helper macro to find OpenSSL eay lib
macro(aux_find_openssl_lib Var)
    set(OPENSSL_EAY_RELEASE OPENSSL_EAY_RELEASE-NOTFOUND)
    set(OPENSSL_EAY_DEBUG OPENSSL_EAY_DEBUG-NOTFOUND)
    set(${Var} ${Var}-NOTFOUND)

    if (MSVC)
        if (OPENSSL_MT)
            set(OpenSSLEay libeay32MT)
        else()
            set(OpenSSLEay libeay32MD)
        endif()

        if (OPENSSL_STATIC)
            set(OpenSSLPostfix lib/VC/static)
        else()
            set(OpenSSLPostfix lib/VC)
        endif()

        find_library(OPENSSL_EAY_RELEASE ${OpenSSLEay} ${OPENSSL_ROOT}/${OpenSSLPostfix} NO_DEFAULT_PATH)
        find_library(OPENSSL_EAY_DEBUG ${OpenSSLEay}d ${OPENSSL_ROOT}/${OpenSSLPostfix} NO_DEFAULT_PATH)
        if(CMAKE_BUILD_TYPE STREQUAL Debug)
            set(${Var} ${OPENSSL_EAY_DEBUG})
        else()
            set(${Var} ${OPENSSL_EAY_RELEASE})
        endif()
    else()
        set(OpenSSLEay libeay32)
        find_library(${Var} ${OpenSSLEay} ${OPENSSL_ROOT}/lib/MinGW NO_DEFAULT_PATH)
    endif()

    if(NOT ${Var})
        message(FATAL_ERROR "Can't find ${OpenSSLEay}")
    else()
        message(STATUS "[OpenSSL] ${${Var}}")
    endif()
endmacro()


##########################################################
## Action!
##########################################################

### First of all find lib.exe
if (MSVC)
    find_program(MSVC_LIB lib)
    if (MSVC_LIB-NOTFOUND)
        message(FATAL_ERROR "Can't find msvc tool: lib.exe")
    endif()
endif()

### Configure Qt
if (BUILD_QT_PLUGIN)
	# Tell CMake to run moc when necessary:
	set(CMAKE_AUTOMOC ON)
	# As moc files are generated in the binary dir, tell CMake
	# to always look for includes there:
	set(CMAKE_INCLUDE_CURRENT_DIR ON)
	
	find_package(Qt5Core REQUIRED)
	find_package(Qt5Sql REQUIRED)
	if (Qt5Core_FOUND)
		message(STATUS "Building with Qt5 support")	
	endif()
endif()


message(STATUS "Searching files")
aux_find_file(SQLITE_SHELL_C shell.c ${SQLITE_SRC_DIR})
aux_find_file(SQLITE_C sqlite3.c ${SQLITE_SRC_DIR})
aux_find_file(CIPHER_CRYPTO_H crypto.h ${CIPHER_ROOT}/src)
aux_find_file(CIPHER_CRYPTO_C crypto.c ${CIPHER_ROOT}/src)
aux_find_file(CIPHER_CRYPTO_OPEN_SSL_C crypto_openssl.c ${CIPHER_ROOT}/src)
aux_find_file(CIPHER_CRYPTO_IMPL_C crypto_impl.c ${CIPHER_ROOT}/src)
aux_find_file(CIPHER_PAGER_C pager.c ${CIPHER_ROOT}/src)
aux_find_file(CIPHER_FUNC_C func.c ${CIPHER_ROOT}/src)
aux_find_openssl_lib(OPENSSL_LIB)
message(STATUS "Searching files - done")


message(STATUS "Preparing sqlite3.c")
set(SQLITESRC_C ${CMAKE_CURRENT_BINARY_DIR}/sqlite3.c)
file(READ ${SQLITE_C} BUF)

## In order to register sqlcipher_export function, we need to replace stuff in sqlite3.c:
file(READ ${CIPHER_FUNC_C} BUF2)
string(REGEX MATCH "\\/[\r\n]void sqlite3RegisterBuiltinFunctions(.*)\\/[\r\n]}[\r\n]" val "${BUF2}")
if (NOT CMAKE_MATCH_1)
    message(FATAL_ERROR "Can't find sqlite3RegisterBuiltinFunctions() in ${CIPHER_FUNC_C}")
endif()
#message(STATUS "Matched sqlite3RegisterBuiltinFunctions(): ${CMAKE_MATCH_1}")
string(REGEX REPLACE "\\/[\r\n]SQLITE_PRIVATE void sqlite3RegisterBuiltinFunctions.*static void setLikeOptFlag" "/\nSQLITE_PRIVATE void sqlite3RegisterBuiltinFunctions${CMAKE_MATCH_1}/\n}\n\nstatic void setLikeOptFlag" BUF "${BUF}")

file(WRITE ${SQLITESRC_C} "${BUF}")
file(APPEND ${SQLITESRC_C} "#include \"crypto.c\"\r\n")
#file(APPEND ${SQLITESRC_C} "#include \"crypto_openssl.c\"\r\n")

message(STATUS "Preparing sqlite3.c - done")


message(STATUS "Preparing crypto.c")
set(CRYPTO_SRC ${CMAKE_CURRENT_BINARY_DIR}/crypto.c)
#set(CRYPTO_OPENSSL_SRC ${CMAKE_CURRENT_BINARY_DIR}/crypto_openssl.c)

### Create crypto.c
file(READ ${CIPHER_CRYPTO_H} BUF)
file(WRITE ${CRYPTO_SRC} "${BUF}")

### Append crypto.c to the new crypto.c
file(READ ${CIPHER_CRYPTO_C} BUF)
string(REGEX REPLACE "#include \"sqliteInt.h\"" "" BUF "${BUF}")
string(REGEX REPLACE "#include \"btreeInt.h\"" "" BUF "${BUF}")
string(REGEX REPLACE "#include \"crypto.h\"" "" BUF "${BUF}")
file(APPEND ${CRYPTO_SRC} "${BUF}")

### Append crypto_openssl.c to the new crypto.c
file(READ ${CIPHER_CRYPTO_OPEN_SSL_C} BUF)
string(REGEX REPLACE "#include \"sqliteInt.h\"" "" BUF "${BUF}")
string(REGEX REPLACE "#include \"btreeInt.h\"" "" BUF "${BUF}")
string(REGEX REPLACE "#include \"crypto.h\"" "" BUF "${BUF}")
file(APPEND ${CRYPTO_SRC} "${BUF}")

### Append crypto part from the pager.c to crypto.c
file(READ ${CIPHER_PAGER_C} BUF)
string(REGEX MATCH "/\\* BEGIN SQLCIPHER \\*/(.*)/\\* END SQLCIPHER \\*/" val "${BUF}")
if (NOT CMAKE_MATCH_1)
    message(FATAL_ERROR "Can't get data from ${CIPHER_PAGER_C}")
endif()
file(APPEND ${CRYPTO_SRC} "${CMAKE_MATCH_1}")

### Append crypto_impl.c to crypto.c
file(READ ${CIPHER_CRYPTO_IMPL_C} BUF)
string(REGEX REPLACE "#include \"sqliteInt.h\"" "" BUF "${BUF}")
string(REGEX REPLACE "#include \"btreeInt.h\"" "" BUF "${BUF}")
string(REGEX REPLACE "#include \"crypto.h\"" "" BUF "${BUF}")
file(APPEND ${CRYPTO_SRC} "${BUF}")

message(STATUS "Preparing crypto.c - done")
message(STATUS "Build shell: ${BUILD_SHELL}")

### Configure targets
set(CMAKE_DEBUG_POSTFIX d)
include_directories(${OPENSSL_ROOT}/include)
include_directories(${SQLITE_SRC_DIR})
include_directories(${CIPHER_ROOT}/src)
include_directories(${QT_QMAKE_EXECUTABLE}/../../../Src/qtbase/src/sql/drivers/sqlite)
include_directories(${QT_QMAKE_EXECUTABLE}/../../../Src/qtbase/include/QtCore/5.1.1)
include_directories(${QT_QMAKE_EXECUTABLE}/../../../Src/qtbase/include/QtCore/5.1.1/QtCore)
include_directories(${QT_QMAKE_EXECUTABLE}/../../../Src/qtbase/include/QtSql/5.1.1)
add_definitions(${CIPHER_DEF})

# cipher lib
add_library(sqlcipher STATIC ${SQLITESRC_C})
target_link_libraries(sqlcipher ${OPENSSL_LIB})

# combine libeay and sqlcipher libs into single sqlcipher.lib
if (MSVC)
    add_custom_command(TARGET sqlcipher POST_BUILD
                       COMMAND ${MSVC_LIB}
                       ARGS "/OUT:$<TARGET_FILE:sqlcipher>" $<TARGET_FILE:sqlcipher> ${OPENSSL_LIB})
endif()

# sqlite tool
if (BUILD_SHELL)
    add_executable(sqlite ${SQLITE_SHELL_C})
    target_link_libraries(sqlite sqlcipher)
    add_dependencies(sqlite sqlcipher)
endif()

# Qt5 sqlcipher plugin
if (BUILD_QT_PLUGIN)
    add_definitions(-DQT_PLUGIN)
    qt5_wrap_cpp(SRC ${QT_QMAKE_EXECUTABLE}/../../../Src/qtbase/src/sql/drivers/sqlite/qsql_sqlite_p.h)
    add_library(qsqlcipher SHARED smain.cpp ${SRC} ${QT_QMAKE_EXECUTABLE}/../../../Src/qtbase/src/sql/drivers/sqlite/qsql_sqlite.cpp)
    target_link_libraries(qsqlcipher ${QT_LIBRARIES} sqlcipher)
    qt5_use_modules(qsqlcipher Core Sql)
    add_dependencies(qsqlcipher sqlcipher)
    install(TARGETS qsqlcipher DESTINATION ${QT_PLUGINS_DIR}/sqldrivers)
endif()
