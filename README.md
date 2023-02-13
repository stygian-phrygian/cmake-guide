CMake Guide
===========
-   [Why?](#why)
-   [How do we build a CMake project?](#how-do-we-build-a-cmake-project)
    -   [What is a CMake project?](#what-is-a-cmake-project)
        -   [CMakeLists.txt](#cmakeliststxt)
        -   [The "source" and "binary" directories](#the-source-and-binary-directories)
        -   [CMakeCache.txt](#cmakecachetxt)
        -   [Configure and generate steps](#configure-and-generate-steps)
    -   [Generate the build pipeline](#generate-the-build-pipeline)
        -   [Cmake flags](#cmake-flags)
            -   [Build type](#build-type)
            -   [Compilation database](#compilation-database)
    -   [Install a CMake project](#install-a-cmake-project)
-   [The CMake Language](#the-cmake-language)
    -   [General CMake syntax](#general-cmake-syntax)
    -   [Commands, Functions, and Macros](#commands-functions-and-macros)
    -   [Variables](#variables)
        -   [Setting variables](#setting-variables)
        -   [Getting variables](#getting-variables)
        -   [Generator Expressions](#generator-expressions)
-   [Modern CMake](#modern-cmake)
    -   [Targets and Properties](#targets-and-properties)
    -   [Property Inheritance](#property-inheritance)
    -   [target\_link\_libraries (the most important command)](#target_link_libraries-the-most-important-command)
    -   [Project Structure](#project-structure)
        -   [project directory](#project-directory)
        -   [build directory](#build-directory)
        -   [include and src directories](#include-and-src-directories)
        -   [app directory](#app-directory)
        -   [tests directory](#tests-directory)
        -   [extern directory](#extern-directory)
        -   [cmake directory](#cmake-directory)
-   [What's in the CMakeLists.txt file?](#whats-in-the-cmakeliststxt-file)
    -   [The project CMakeLists.txt](#the-project-cmakeliststxt)
    -   [The src CMakeLists.txt](#the-src-cmakeliststxt)
    -   [The apps CMakeLists.txt](#the-apps-cmakeliststxt)
    -   [The tests CMakeLists.txt](#the-tests-cmakeliststxt)
-   [How does CMake import dependencies?](#how-does-cmake-import-dependencies)
    -   [Git submodules](#git-submodules)
        -   [Add a git submodule](#add-a-git-submodule)
        -   [Recursively update our project's submodules](#recursively-update-our-projects-submodules)
        -   [Alert our project where the git submodule dependency is](#alert-our-project-where-the-git-submodule-dependency-is)
    -   [FetchContent module](#fetchcontent-module)
        -   [Call the FetchContent module](#call-the-fetchcontent-module)
    -   [ExternalProject module](#externalproject-module)
        -   [Call the ExternalProject module](#call-the-externalproject-module)
    -   [find\_package() command](#find_package-command)
        -   [Call find\_package()](#call-find_package)
        -   [Write find modules](#write-find-modules)
-   [How does CMake run tests?](#how-does-cmake-run-tests)
    -   [ctest](#ctest)
    -   [Copy test data](#copy-test-data)
-   [How do we export a CMake project?](#how-do-we-export-a-cmake-project)
    -   [Write config modules](#write-config-modules)
-   [Sources](#sources)

Why?
====

CMake is a language that generates a build pipeline. That's it. But why
should we care?

Because, building *large* C++ projects is **very** difficult.

Problems with traditional C++ build systems:
-   Where are the dependency headers and their shared objects?
-   What about on a different OS?
-   Do those dependencies also require dependencies? How do we download them?
-   Which compilation flags do they require and what must the compiler know?
-   Can we integrate multiple programming languages?
-   Can we compile for a different architecture?
-   How do we export our project for use by other projects?

A makefile *won't* scale.

CMake solves all of these problems.

CMake:
1.  generates build pipelines in a cross platform way
2.  is extensible (is scriptable)
3.  is the defacto C++ build tool

There are recent competitors in the realm of C++ package managers (Vcpkg
and Conan) but CMake still dominates.

How do we build a CMake project?
================================

Let's say we just downloaded a CMake project. Now what? How do turn our
source into binaries?

Because CMake is *not* a build system, we must first generate our build
pipeline. *Then*, we build.

But, let's clearly define what a CMake project is and what it expects.

What is a CMake project?
------------------------

A CMake project is a directory with a CMake language file called
CMakelists.txt. That's *technically* all a cmake project is. However,we
*strongly* recommend a specific subdirectory organization described further along.

Let's explain the expectations of a CMake project further.

### CMakeLists.txt 

The CMakeLists.txt file, written in CMake, describes our project's
binary targets (executables, libraries, etc) plus their dependencies.
Much more will be said about it. Suffice it to say, we often edit this file.

### The \"source\" and \"binary\" directories

To CMake:
-   a source directory is the location of a CMakeLists.txt file
-   a binary directory is the output location of our binary targets,
    invariably called build, and where the CMakeCache.txt file resides

Here, "source" refers to CMake source code (in other CMakeLists.txt
files). Our C++ source code is located in other subdirectories,
frequently with their own CMakeLists.txt as well. This implies that
there are multiple source directories, although CMake distinguishes
between the root source directory and the currently parsed source
directory.

Our binary directory is *separate* from our project's source code. This
is called an "out of source" build and allows us a simple way to clean
up our project. We just delete the binary directory.

### CMakeCache.txt 

Crucially, the binary directory contains a file called CMakeCache.txt.
This is generated *only* once when we call cmake.

Obviously, it caches build variables. They are plain text so we can
directly read them with a editor. Generally we do not edit this file
directly. We either generate a build pipeline or override existing cache
variables as flags to our cmake command.

Storing them allows us to have multiple build configurations (multiple
binary directories) which is *very* common. Maybe we want a debug build
or a release build? Maybe a cross compilation?

### Configure and generate steps

Creating a build pipeline occurs in 2 distinct steps:
1.  the configure step, then
2.  the generate step

During the configure step, CMake parses the CMakeLists.txt in the source
directory given it. If all this succeeds, we move to the next step.
Note, CMakeLists.txt often includes subdirectories with their own
CMakeLists.txt

During the generate step, CMake chooses a build system then generates it
into the binary directory. By default, CMake guesses which build system
to employ based off the current platform. If we want, we may specify
this. Here, CMake writes the CMakeCache.txt file.

The distinction between the configure and generate steps matters because
certain build information is *only* available during the generate step.
However, it\'s sometimes useful for the CMakeLists.txt file to access
generate step information *during* the configure step. The CMake
language allows this, in a very controlled fashion briefly touched
below, with generator expressions.

Generate the build pipeline
---------------------------

Assuming we're in our project's top level source directory (it has a
CMakeLists.txt), the usual procedure to create a build pipeline is:

```
mkdir build # create our binary directory, usually called "build"
cd build    # cd to it
cmake ..    # run cmake, passing our source directory
```

Alternatively, we may achieve the above in one command:

```
cmake -S <source-directory> -B <binary-directory>
```

And, to actually build the targets (assuming we're within the binary
directory)

```
make # or whichever platform specific build tool was chosen
```

Or, if we prefer to remain in the source directory

```
cmake --build <binary-directory>
```

Suppose we modified some source C++ files, do we have to regenerate the
build pipeline? No. CMake figures it out. We only run cmake if we wish
to generate a differently configured binary directory or perhaps
override existing cached variable configuration.

### CMake flags

We may pass *many* flags to the cmake command, hundreds. Plus, we may
craft our own. CMake stores them as cache variables in the CMakeCache.txt.
Defaults are provided unless we explicitly override. For introductory
purposes, we'll constrain ourselves to just a few useful developer variables.
However, all of them follow similar syntax

```
cmake -S . -B build/<someBuildConfiguration> -D <someVariable>=<someValue>
```

#### Build type

Sometimes we prefer to compile our targets with debug information
included. Sometimes we prefer to optimize for speed or binary size
instead. We may direct CMake to alter our build type with the aptly
named, CMAKE\_BUILD\_TYPE variable.

By default, CMake ships with the following available build type
configurations:
-   Debug  - include debug information
-   Release  - optimize for speed
-   RelWithDebInfo  - optimize for speed with debug information
-   MinSizeRel  - optimize for size and speed

Other build types are possible with more configuration.

Here\'s an example usage

```
cmake -S . -B build/release -D CMAKE_BUILD_TYPE=Release  
```

#### Compilation database

We may skip thisi unless our IDE lacks C++ semantic understanding (auto-completion or linting).

Compilation databases are meta data about our project's source code.
They're *not* necessary for building the project however they are often
*useful* for development.

IDEs communicate to a language server to glean information for
auto-completion and linting purposes. The industry standard C++ language
server is clangd. It consumes a specific format of compilation database
called compile\_commands.json. Fortunately, CMake easily generates them.

Here's an example usage

```
cmake -S . -B build -D CMAKE_EXPORT_COMPILE_COMMANDS=ON 
```

The above will *flood* json and CMake files around our project so it's
worth investigating how .gitignore files work. Moreover, we may have to
alert our IDE to the location of the generated compilation database.

Install a CMake project
-------

Assuming CMake created a build pipeline *and* we built, we should have
binary artefacts scattered about our binary directory. Installing them
into our system is simple enough.

From the binary directory, run

make install 

Or, if we have CMake \> version 3.15, from the source directory

```
cmake --install . 
```

CMake offers us a shortcut flag for configuring the build type

```
cmake --install . --config Release 
```

The CMake Language
==================

Before we begin editing CMakeLists.txt files, we'll have to understand
some basics of the CMake language.

The CMake language has a number of included commands, variables,
functions, and macros. These are roughly equivalent to any modern
programing language's repertoire.

General CMake syntax
--------------------

Most CMake language constructs resembles the following:

command(args) where:

    command is:\
            a CMake command\
            a function\
            a macro\
            control flow\
            loop\
            etc

   args is:\
            anything (but really a string)

To see all commands:\
    cmake --help-command-list 

Commands, Functions, and Macros
-------------------------------

Some example commands:

```
add_library(myLib src/mylib.cpp)
target_include_directories(myLib PUBLIC include/myLib.h)
add_executable(myApp src/main.cpp)
target_link_libaries(myApp myLib)
```

We can define our own commands. User defined commands are called
functions or macros. The only difference between functions and macros is
scope. Macros have no variable scope thus are to be cautiously employed.

An example function definition:

```
# create a function "print_list" which (obviously) prints a list
function(print_list my_list)
    foreach(var IN LISTS my_list)
           message("${var}")
    endforeach()
endfunction()
```

CMake offers us a few variables to access arguments to our function:

ARGC is the count of arguments\
ARGV is the list of arguments\
ARGN  is the list of arguments (we haven\'t assigned names to)\
ARG0 , ARG1 , ARG2 , etc are the 1st, 2nd, 3rd, etc arguments

Variables
---------

### Setting variables

Variables come in 3 types: booleans, strings, and lists. However,
they're really all strings. Their scope is for the duration of the file
or function they're defined within.

```
# these are strings
set(my_string hello)
set(also_my_string "hello world")

# these are lists (of strings)
set(my_list 0 1 2 3)

# a string delimited by semicolons is a "list"
set(also_my_list "0; 1; 2; 3; 4")

# these are bools (again really just strings)

# evaluates to true:
# 1, on, yes, true, y

# evaluates to false:
# 0, off, no, false, n, NOTFOUND, -NOTFOUND

set(my_true_bool true)
set(also_my_true_bool yes)

set(my_false_bool false)
set(also_my_false_bool no)

# because bools are common there is a shortcut

# if this is evaluated in CMake project mode (not script mode)
# it sets a cache variable
option(SOME_FLAG "information about this flag" OFF)
```

There are 2 more types of variable scope: cache, and environmental.

Cache variables write to the CMakeCache.txt contained in the binary
directory. Note, a cache variable is set *only* when its name is *not
already present* in the CMakeCache.txt file. To force it to set, we have
to pass the FORCE argument.

Environmental variables exist briefly while the CMake process runs
during the configuration and generation phases of our project. Contrary
to regular shell environment variables, they do not persist after these
phases. Use with caution.

```
# normal
set(the_answer 42)

# cache (does not overwrite existing value **if** present)
set(my_cache_var "VALUE" CACHE STRING "Description")

# cache (overwrites existing value always)
set(my_cache_var "VALUE" CACHE STRING "Updated Description" FORCE)                                                             |

# environmental
set(ENV{some_env_var} "whatever")
```

There is another type of variable called a *property* whose scope is
attached to a *target*. We will discuss them later in the modern CMake
section.

### Getting variables

Variables are evaluated with a variable expansion syntax:

```
${some_var}
```
We may surround variable evaluation in double quotes to
avoid misinterpreting its "type":
```
"${some_var}"
```

Environmental variables are evaluated with a slightly different syntax:
```
$ENV{some_env_var}
```

To see all variables:\
    cmake --help-variable-list

### Generator Expressions

As already stated, CMake runs in 2 major phases: configuration and
generation. Sometimes we desire information that is *only* available
during CMake's generation phase. We acquire this information via
generator expressions. These possess a strange syntax and many
evaluation forms but resemble the following

```
$<...>
```

where ... is something that evaluates with configuration information.

A more concrete example

```
compile_definitions(
    $<$<CONFIG:Debug>:-DDEBUG_LEVEL=2>
    $<$<CONFIG:Release>:-DDEBUG_LEVEL=0>
)
```

This has multiple (and nested) generator expressions.

If `Debug` exists, return `1`.
Otherwise, return `0`.

`$<CONFIG:Debug>`

If `Debug` is defined (returned `1`), return the string `DEBUG_LEVEL=2`.
Otherwise, return `""`

`$<$<CONFIG:Debug>:-DEBUG_LEVEL=2>`

If `Release` exists, return `1`.
Otherwise, return `0`.

`$<CONFIG:Release>`

If `Release` is defined (returned `1`), return the string `DEBUG_LEVEL=0`.
Otherwise, return `""`.

`$<$<CONFIG:Release>:-DEBUG_LEVEL=0>`

We know, it's a bizarre inconsistent syntax, however unavoidably
useful. As is observed above, they often function similar to ternary
expressions from the C languages.

To finish our swift tour of the CMake language, this example demonstrates 
functions and variables.

```

# create a function to print element in a list if they evaluate to true
function(print_list my_list)
    foreach(var IN LISTS my_list)
        if(var)
            message("${var}")
        endif()
    endforeach()
endfunction()

# create a "list" variable that can be interpreted as bools
set(list_of_bools 0 1 2 3 on off yes no true false y n NOTFOUND Library-NOTFOUND)                                                  |

message(list_of_bools)
message(${list_of_bools})
# 0123onoffyesnotruefalseynNOTFOUNDLibrary-NOTFOUND

message("${list_of_bools}")
# 0;1;2;3;on;off;yes;no;true;false;y;n;NOTFOUND;Library-NOTFOUND

print_list("${list_of_bools}")
# 1
# 2
# 3
# on
# yes
# true
# y

```

Modern CMake
============

In prior CMake, versions less than around version 3.0, developers often
specified dependencies via global variables. Naturally, this threatened
extensibility as project scope swelled. Library X requires these build
flags and library Y requires something else. Brittle interfaces quickly
spread. Can we do better?

Yes.

Targets and Properties
----------------------

"Modern CMake" as it is called, versions greater than around 3.0,
**encapsulates** dependencies with *targets* and *properties*.

A target is a product of our CMake build: an executable, a library, a
test. It's what we want our generated build *to build*. How do we
specify what it requires to build? Properties.

A property is information *scoped to a target*. CMake predefines many
properties which we attach to targets via commands. Properties represent
target information such as: compilation flags, linking flags,
preprocessor flags, C++ standard, include directories, etc.

Targets are (usually) created by the following commands:
```
add_executable()
add_library()
add_test()
```
Properties are (usually) attached to a target with the following
commands:
```
target_link_libraries()
target_include_directories()
target_compile_definitions()
target_compile_features()
target_compile_options()
```

We may get/set any property for a target directly with the following
commands (but higher level commands are preferred such as the above):
```
set_target_properties()
get_target_property()
```

To see all properties:\
    cmake --help-property-list

To exploit a metaphor from Object Oriented Programming, one can think of
targets as *objects* and properties as *member variables*. And, just
like member variables, inheritance is possible.

Property Inheritance
--------------------

When targets link to other targets, properties may be inherited. CMake
specifies this through keywords arguments: PRIVATE, INTERFACE, or PUBLIC.

As an example, suppose we have a library target, myLib, that requires 3 other
library targets: fmt, json, and network. Those who link to our library, myLib,
should not inherit access to fmt or json but should inherit access to
network.

```
target_link_libraries(
    myLib
    PRIVATE
    fmt
    json
    PUBLIC
    network
)
```

Now, we link someOtherLib to myLib. Note, someOtherLib  inherits access
to network (because it was specified PUBLIC) but neither fmt nor
json (because they were specified PRIVATE)

```
target_link_libraries(someOtherLib myLib)
```

So what's this INTERFACE keyword argument? Outside of exporting header
only library targets, it's rarely used.

Internally, CMake *properties* come in *pairs* which represent their
inheritance scopes. CMake calls them interface or private.
Interface *properties* are inherited and their names prefixed with
INTERFACE_. Private *properties* are not inherited nor is their name
prefixed with anything.

Thus, there are 3 possible cases of *property* inheritance. To determine
which to use, we ask ourselves the following:
-   Is this dependency needed by me *but not* needed by dependents? Use
    PRIVATE (only define private property)
-   Is this dependency *not* needed by me *but* needed by dependents?
    Use INTERFACE (only define interface property)
-   Is this dependency needed by me *and* needed by dependents? Use
    PUBLIC (define both private and interface properties)

target\_link\_libraries (the most important command)
----------------------------------------------------

We want to emphasize how important this command is.

First, consider the non-modern way dependencies were managed **(do not do this):**
-   target A depends on variable B's headers
-   target A depends on variable B's libraries
-   target A depends on variable B's compile flags
-   target A depends on variable B's *whatever*

This scatters B's dependencies in variables that A must now know about.
Who else knows about them? Who knows? It increases the surface area of
dependencies and thus coupling bandwidth.

Instead, we *only* want to say:
-   target A depends on target B

Because of property inheritance, we may say this. There is a single
CMake command which says this. We saw it above.

`target_link_libraries(A B)`

When creating a library target, we set its properties with controlled
access. Then, its dependents automatically inherit them. Simple.
Modular. **Do this.**

For historical reasons, this command accepts many types of *non-target*
arguments: raw build flags, shared objects, static libraries, etc. **Do
not** use non-target arguments. **Prefer** *target* arguments lest we
break encapsulation.

Project Structure
-----------------

If Modern CMake encourages modularity, it's further emphasized by how
we organize our project directory structure. A modern CMake project
should resemble the following (names vary):

```
├── CMakeLists.txt
├── README.md
├── apps
│   ├── CMakeLists.txt
│   └── app.cpp
├── build
├── cmake
│   └── FindSomeLib.cmake
├── extern
├── include
│   └── modern
│   └── lib.hpp
├── src
│   ├── CMakeLists.txt
│   └── lib.cpp
└── tests
├── CMakeLists.txt
└── testlib.cpp
```

### `project` directory

At the top of the project directory is a single CMakeLists.txt which
organizes everything (explained later how).

### `build` directory

The build directory contains the project output ie. binary artifacts,
what CMake calls targets. These targets are: executables, libraries,
tests (the stuff we actually *build*). Note, this directory is
intentionally separate from any source code which allows us to trivially
remove it.

### `include` and `src` directories

If our project creates libraries, meaning it has library targets
(shared/static library binary artifacts), we have the include and src
directories. These respectively contain the header and source files for
our *library* targets. Note, these directories *only* contain what *we*
wrote *not* 3rd party content (contained in extern or found by CMake
modules). Also, note that src has a CMakeLists.txt file while include
does not. To ease project installation, we separate the public header
and source directories.

### `app` directory

If our project creates executables, meaning it has executable targets
(executable binary artifacts), we have the apps directory. As this is
not a library target (meaning we have no need to copy headers) the apps
directory contains *both* the headers *and* source for our executable
targets. These are together because we won't be copying headers from
this directory.

### `tests` directory

If our project creates tests, meaning it has test targets (which are
really executable targets as well but hooked into CMake\'s testing
system entitled ctest), we have the tests directory.

### `extern` directory

Our project very likely depends on 3rd party libraries. There are a
variety of ways to link to these which *greatly* depends on their
location (on or off system) and whether or not they're a properly
configured CMake project (dubious). Assuming our 3rd party libraries are
git based CMake projects, we place their git submodules in the extern
directory.

### `cmake` directory

Alas, it's very likely we will depend on 3rd party libraries that *are
not* CMake projects (raw headers and static/shared objects on our
system). How do we import these? Answer: CMake modules. A CMake script
within the CMAKE\_MODULE\_PATH is called a CMake module. CMake ships
with many modules which find libraries and create imported target
libraries. But, it's likely that we'll have to write our own. We'll
put these, and other related modules, in the cmake directory.

To see all included modules:\
    cmake --help-module-list 

What's in the CMakeLists.txt file?
==================================

A CMakeLists.txt file describes *what* to build (the targets) and *how*
(the dependencies via properties). Therefore, its contents varies.
However, utilizing the recommended project structure (prior described),
we will tour the usual contents of a CMakeLists.txt file per directory.

The project CMakeLists.txt
--------------------------

*defines:*
-   minimum CMake version
-   project name
-   3rd party dependencies (how this is accomplished varies, see section
    regarding how we import dependencies)
-   target subdirectories

*example:*
```
# Minimum CMake Version
--------------------------------------------------------------------
cmake_minimum_required(VERSION 3.15)

# Project Name (bonus: version, description, languages)
--------------------------------------------------------------------
project(
    ModernCMakeExample
    VERSION 0.1
    DESCRIPTION "An example project with CMake"
    LANGUAGES CXX
)
# 3rd Party Dependencies
--------------------------------------------------------------------
# add FetchContent CMake module (this downloads non-local CMake projects)
include(FetchContent)

FetchContent_Declare(
fmtlib
GIT_REPOSITORY https://github.com/fmtlib/fmt.git
GIT_TAG 5.3.0)
FetchContent_MakeAvailable(fmtlib)

find_package(Boost REQUIRED)

# Target Subdirectories
--------------------------------------------------------------------
# our library targets
add_subdirectory(src)

# our executable targets
add_subdirectory(apps)

# our test targets
add_subdirectory(tests)
```

The src CMakeLists.txt 
----------------------

*defines:*
-   library targets (name and source)
-   library properties (include directory and libraries, optionally other properties)

*example:*
```
# Library Targets
--------------------------------------------------------------------
add_library(
    # name
    modern_library
    # source
    lib.cpp
)

# Library Properties
--------------------------------------------------------------------
# our includes, PUBLIC allows dependents access
target_include_directories(modern_library PUBLIC ../include)

# 3rd party library
target_link_libraries(modern_library PRIVATE Boost::boost)

# other properties
target_compile_features(modern_library PUBLIC cxx_std_11)
```

The apps CMakeLists.txt 
------------------------

*defines:*
-   executable targets (name and source)
-   executable properties (libraries or other properties)

*example:*
```
# Executable Targets
--------------------------------------------------------------------
add_executable(app app.cpp)

# Executable Properties
--------------------------------------------------------------------
target_link_libraries(app PRIVATE modern_library fmt::fmt)
target_compile_features(app PRIVATE cxx_std_17)
```

The tests CMakeLists.txt 
-------------------------

*defines:*
-   optional: 3rd party dependencies (ie. the testing framework) which
    may be defined at the project scope or locally here
-   executable targets (name and source)
-   executable properties (our library, testing framework)
-   test targets (register executable with ctest)

*example:*
```
# optional: 3rd party library
--------------------------------------------------------------------
FetchContent_Declare(
    catch
    GIT_REPOSITORY https://github.com/catchorg/Catch2.git
    GIT_TAG v2.13.6
)
# Adds library target: Catch2::Catch2
FetchContent_MakeAvailable(catch)

# Executable Targets
--------------------------------------------------------------------
add_executable(testlib testlib.cpp)

# Executable Properties
--------------------------------------------------------------------
# in this case, we're linking to Catch2, which we included above
target_link_libraries(testlib PRIVATE modern_library Catch2::Catch2)
target_compile_features(testlib PRIVATE cxx_std_17)

# Test Targets
--------------------------------------------------------------------
# register test with ctest to run it
add_test(NAME testlibtest COMMAND testlib)
```

How does CMake import dependencies?
===================================

We have the following choices:
-   git submodules
-   the FetchContent module
-   the ExternalProject module
-   the find\_package() command

Each has caveats, but generally git submodules or the FetchContent
module are the easiest (least surprising) choices.

The following examples assume the recommend directory structure.

Git submodules
--------------

Perhaps the easiest method, simply put git submodules in our project and
build them from source.

We prefer the dependencies to be CMake projects though it's possible
(with hassle) they aren't. Of course, git must be locally *available*.
When cloning our project we must now also recursively update submodules
but this may be scripted away.

### Add a git submodule

From within our project directory, for each dependency, run the
following

```
git submodule add https://github.com/someAuthor/someLibrary extern/someLibrary 
```

### Recursively update our project's submodules

From within our project directory, run the following

```
git submodule update --init --recursive  
```

However, the above must be called before building which is a lot to
expect of our users. Instead, we inform CMake to automatically update git
submodules for us.

```
if(GIT_FOUND AND EXISTS "${PROJECT_SOURCE_DIR}/.git")
    # Update submodules as needed
    option(GIT_SUBMODULE "Check submodules during build" ON)
    if(GIT_SUBMODULE)
        message(STATUS "Submodule update")
        execute_process(
            COMMAND ${GIT_EXECUTABLE} submodule update --init --recursive
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
            RESULT_VARIABLE GIT_SUBMOD_RESULT
        )
        if(NOT GIT_SUBMOD_RESULT EQUAL "0")
            message(
                FATAL_ERROR
                "git submodule update --init --recursive failed with ${GIT_SUBMOD_RESULT}, please checkout submodules"
            )
        endif()
    endif()
endif()

if(NOT EXISTS "${PROJECT_SOURCE_DIR}/extern/repo/CMakeLists.txt")
    message(
        FATAL_ERROR
        "The submodules were not downloaded! GIT_SUBMODULE was turned off or failed. Please update submodules and try again."
    )
endif()
```

### Alert our project where the git submodule dependency is

In our top level project CMakeLists.txt file, we write:

```
add_subdirectory(extern/someGitProject)
```

FetchContent module
-------------------

Perhaps the easiest method that modern CMake supports *out of the box* since 
version 3.14. 

The FetchContent module downloads dependencies at **configure** time.
Thus, our project may use this information for commands like
add\_subdirectory(), include(), or file(). We prefer that the
dependencies are CMake projects but this not strictly required (though
will again incur hassle). Since CMake 3.24, FetchContent may employ
find\_package() with the FIND\_PACKAGE\_ARGS parameter.

### Call the FetchContent module

We include the FetchContent module, declare dependencies, make them
available, then use them as imported targets.

The following example fetches Catch2 and GoogleTest.

```
# include the FetchContent module
include(FetchContent)

# declare dependencies
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG 703bd9caab50b139428cea1aaff9974ebee5742e #release-1.10.0
)

FetchContent_Declare(
    Catch2
    GIT_REPOSITORY https://github.com/catchorg/Catch2.git
    GIT_TAG de6fe184a9ac1a06895cdd1c9b437f0a0bdf14ad # v2.13.4
)

FetchContent_MakeAvailable(googletest Catch2)

# use the imported target that GoogleTest created, check the project to determine its target
add_executable(ThingUnitTest thing_ut.cpp)
target_link_libraries(ThingUnitTest GTest::gtest_main)
```

ExternalProject module
----------------------

The ExternalProject module is similar to the FetchContent module but
older, more complex, and *crucially* downloads libraries only when we
**build** (run make or whichever our platform demands). *Thus*,
configuration step information is *unavailable*. It's usually employed
by what the CMake community knows as "super builds" or builds that
chain on each other. Generally, git submodules or the FetchContent
module are easier to use however they expect CMake dependencies which
ExternalProject does not.


### Call the ExternalProject module

Our example pulls a zipped CMake project. This is only rudimentary,
consult the documentation for the myriad of potential uses.

```
include(ExternalProject)
ExternalProject_add(
    someTarget
    URL https://github.com/someAuthor/someProject/v2.1.0.zip
    CMAKE_ARGS
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
    -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
)
```

find\_package() command
-----------------------

The find\_package() command differs from the prior choices insofar as it
*does not download anything.* Rather, find\_package() searches our local
system. Internally, it calls a CMake module to acquire the dependency
and (hopefully) assemble it into imported targets. The arguments to
find\_package() determine the search mode it will employ, namely module
vs config mode.

Module mode is generally for *non-CMake project* dependencies that
require CMake *modules* to *find* them, known as find modules.

Config mode is for *CMake project* dependencies that are *configured*
with CMake package config files. These files export their targets
names, version information, and dependencies.

Optimally, our call to find\_package() should just work, preferably
using config mode which grants us more information. But, sometimes the
dependency didn't export a config (which should be considered a bug) or
CMake didn't already possess a find module for it. Writing config
modules is the purview of the dependency owners. However, albeit less
than preferred, we may write a find module.

### Call find\_package() 

In any search mode the call to find\_package() looks identical.

```
find_package(GTest REQUIRED) 
```

### Write find modules

Assuming our desired dependency did not ship with package config files
nor CMake shipped with a find module, we must write the find module
ourselves.

Our find module is a CMake module (script) which will set up an imported
library target from local 3rd party headers and libraries. It *must be*
named Find\<PackageName\>.cmake 

So, if our dependency is called SomeLib our find module will be called
FindSomeLib.cmake and we will call it like so

```
find_package(FindSomeLib) 
```

We'll place our find module in the cmake directory and inform CMake of
its location by updating the CMAKE\_MODULE\_PATH variable

```
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/")
```

Now we may write it.

Our find module will
-   add search locations
-   set header location in a cache variable
-   set library location in a cache variable
-   force our find module to behave in a standard way
-   hide cache variables from displaying in the GUI
-   add *imported* library target

As an example, let's write a find module for LibImagePipeline, put it
into the cmake directory, and call it FindLibImagePipeline.cmake 

```
# Add search locations
--------------------------------------------------------------------
# Because the find_path() command does not have default search locations
# we must feed it some.

# Append the GNU install directories to CMAKE_INSTALL_INCLUDEDIR
include(GNUInstallDirs)

# Set the header location in a cache variable
--------------------------------------------------------------------

# On success, set location in LIBIMAGEPIPELINE_INCLUDE_DIR
# Otherwise, set LIBIMAGEPIPELINE_INCLUDE_DIR-NOTFOUND

find_path(
    LIBIMAGEPIPELINE_INCLUDE_DIR
    NAMES Pipeline.hpp
    HINTS ${CMAKE_INSTALL_INCLUDEDIR}
)
# Set the library location in a cache variable
--------------------------------------------------------------------

# On success, set location in LIBIMAGEPIPELINE_LIBRARY
# Otherwise, set LIBIMAGEPIPELINE_LIBRARY-NOTFOUND
# Note, we omit the library extensions .so .dll .a
# Also, we do not need to provide HINTS (though we may)
find_library(
    LIBIMAGEPIPELINE_LIBRARY
    NAMES LibImagePipeline
)

# Force our find module to behave in a standard way
--------------------------------------------------------------------

# By "standard way", we mean our find module should:
# set <PackageName>-FOUND (on success)
# handle the REQUIRED, QUIET, and version related arguments to the find_package() command
# This is a command which does this for us
include(FindPackageHandleStandardArgs)
find_package_handle_standard_args(
    libImagePipeline
    DEFAULT_MSG
    LIBIMAGEPIPELINE_LIBRARY
    LIBIMAGEPIPELINE_INCLUDE_DIR
)

# Hide cache variables from displaying in the GUI                 |
--------------------------------------------------------------------

# In a CMake GUI, variables marked as "advanced" do not show up  unless explicitly requested
# This step is not strictly necessary but nice

mark_as_advanced(LIBIMAGEPIPELINE_LIBRARY LIBIMAGEPIPELINE_INCLUDE_DIR)

# Add imported library target
--------------------------------------------------------------------

if(LIBIMAGEPIPELINE_FOUND AND NOT TARGET libImagePipeline::libImagePipeline)
    add_library(libImagePipeline::libImagePipeline SHARED IMPORTED)
    set_target_properties(
        libImagePipeline::libImagePipeline
        PROPERTIES

        INTERFACE_INCLUDE_DIRECTORIES
        "${LIBIMAGEPIPELINE_INCLUDE_DIR}"

        IMPORTED_LOCATION
        ${LIBIMAGEPIPELINE_LIBRARY}
    )
endif()
```

How does CMake run tests?
=========================

CMake's provided test runner is called ctest.

A test runner (obviously) executes the project's unit tests. The
advantage isn't obvious until a project grows significantly large. We
may script test runners or ask them to run only tests matching a certain
regex.

ctest 
------

To use ctest we must:
1.  write unit tests (varies by framework)
2.  add these as executable targets
3.  register these executable targets as tests

The add\_executable() command will create targets for us. We feed these
targets to add\_test() so ctest knows where and how to run them.

To run ctest we simply call it from the binary directory (presumably
build). By default, it will run *everything* registered as a test. Flags
narrow this down.

With flag -R we select specific tests by a regex match.
With flag -VV we ask for verbose output.

`ctest -VV -R SomeTestSuite`

Copy test data
--------------

Often, unit tests read test data. Our test data may be near the source
code location but CMake will *not* automatically copy it to the runtime
location (somewhere in the binary directory).

With the file() command, we may copy files from the CMake source
directory to the CMake binary directory.

```
file(
    COPY
    some/local/source_directory/data/test_data_0.json
    DESTINATION
    ./data
)
```

How do we export a CMake project?
=================================

Exporting a CMake project means future consumers have access to the following:
-   the binary artefacts (shared and static objects)
-   the public headers
-   the target names
-   the version
-   the dependencies

The binary artefacts and headers are simple to copy. The rest are not.

Through the creation of CMake modules, known as config modules, we export the 
above project information. This process is somewhat clunky, fortunately we may 
direct CMake to generate most of this for us.

Specifically, we want 3 modules:
-   a *config target module* (contains the project's target name information)
-   a *config version module* (contains the project's version information)
-   a *config module* (consumes the above information and specifies which dependencies our project requires)

These modules are then copied into the system library path.

When a future consumer of our project calls find\_package(), in config mode, 
it will directly call our config module. Then, our config module calls the 
target and version modules.

### Write config modules

We have an aptly named command for copying everything into system
locations; the install() command. We feed it a destination path and it
copies. But, instead of hardcoding absolute paths we will exploit
variables. This asks CMake to figure out the best system paths as well
as allows niche overrides (a package maintainer might want an unusual install 
location instead).

We want our config modules to accomplish the following:
-   copy the binary artefacts
-   save the target export set
-   copy the config target module (generated for us)
-   copy the headers
-   write the config module (**not** generated for us)
-   copy the config version module (generated for us)
-   copy the config module

We add the following to our project's CMakeLists.txt and then manually write 
our config module to import what's generated.

```
# import default path prefixes for systems, works even on windows despite the "Gnu" in the module name
include(GnuInstallDirs)

# copy the target artefacts
--------------------------------------------------------------------
install(
    # which targets we want to install
    TARGETS myLibraryTarget1 myLibraryTarget2
    # save their target export set (for use below)
    ----------------------------------------------------------------
    EXPORTS MyLibraryTargets

    # windows shared libraries (.dll)
    RUNTIME DESTINATION ${CMAKE_INSTALL_BIN_DIR}
    # non-windows shared libraries (.so)
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIB_DIR}
    # all platforms static libraries
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIB_DIR}

    # set the targets include destination to"include"
    # note, this *does not* install the include directory (handled below)
    INCLUDES DESTINATION include
)

# copy the config target module
--------------------------------------------------------------------
install(
    # consume target export set
    EXPORT MyLibraryTargets

    # write it into a file usually named <Library>Targets.cmake
    File MyLibraryTargets.cmake

    # use namespace (best practice)
    NAMESPACE MyLibrary::

    # store this file, usually in cmake directory of system lib
    DESTINATION ${CMAKE_INSTALL_LIB_DIR}/cmake/MyLibrary
)

# copy the target headers
--------------------------------------------------------------------
install(
    DIRECTORY include/MyLibrary
    DESTINATION include
)

# ask CMake to generate the config version module for us
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    # write it into a file usually named <Library>ConfigVersion.cmake
    MyLibraryConfigConfigVersion.cmake
    # with this version
    VERSION ${MyLibrary_VERSION}
    # and this backwards compatibility
    COMPATIBILITY SameMajorVersion
)


# write config module (shown after this example)
--------------------------------------------------------------------

# copy config version module
--------------------------------------------------------------------
# copy config module
--------------------------------------------------------------------
install(
    # the config version module we generated earlier
    # the config module which we wrote ourselves
    FILES MyLibraryConfigVersion.cmake MyLibraryConfig.cmake
    # store it, usually in cmake directory of system lib
    DESTINATION ${CMAKE_INSTALL_LIB_DIR}/cmake/MyLibrary
)
```

Now, we write our config module file, usually known as
\<Library\>Config.cmake, which finds our project dependencies and
rebuilds it into imported targets

```

# find project dependencies
include(CMakeFindDependencyMacro)
find_dependency(SomeDependency 1.0)

# rebuild targets
include("${CMAKE_CURRENT_LIST_DIR}/MyLibraryTargets.cmake)
```

Sources
=======

An Introduction to Modern CMake

https://cliutils.gitlab.io/modern-cmake/

An Introduction to modern CMake for beginners

https://www.internalpointers.com/post/modern-cmake-beginner-introduction

CppCon 2017: Mathieu Ropert "Using Modern CMake Patterns to Enforce a Good Modular Design"

https://www.youtube.com/watch?v=eC9-iRN2b04

C++Now 2017: Daniel Pfeifer "Effective CMake"

https://www.youtube.com/watch?v=bsXLMQ6WgIk

CppCon 2018: Mateusz Pusz "Git, CMake, Conan - How to ship and reuse our C++ projects"

https://www.youtube.com/watch?v=S4QSKLXdTtA

Effective Modern CMake

https://gist.github.com/mbinna/c61dbb39bca0e4fb7d1f73b0d66a4fd1

CMake part 1: It is a programming language!

https://iamsorush.com/posts/cpp-cmake-essential/

CMake part 3: create a config file to be found by find\_package()

https://iamsorush.com/posts/cpp-cmake-config/

CMake Primer

https://llvm.org/docs/CMakePrimer.html

It's Time To Do CMake Right

https://pabloariasal.github.io/2018/02/19/its-time-to-do-cmake-right/

Modern CMake Is Like Inheritance

https://kubasejdak.com/modern-cmake-is-like-inheritance

CMake line by line - using a non-CMake library

https://dominikberner.ch/cmake-find-library/

Tutorial: Easy dependency management for C++ with CMake and Git

https://www.foonathan.net/2016/07/cmake-dependency-handling/

Tutorial: Preparing libraries for CMake FetchContent

https://www.foonathan.net/2022/06/cmake-fetchcontent/

How to Build a CMake-Based Project

https://preshing.com/20170511/how-to-build-a-cmake-based-project/

Deep CMake for Library Authors

https://www.youtube.com/watch?v=m0DwB4OvDXk
