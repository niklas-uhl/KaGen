################################################################################
# app/CMakeLists.txt
#
# CMake file for generator app
#
# Copyright (C) 2016-2017 Sebastian Lamm <lamm@ira.uka.de>
#
# All rights reserved. Published under the BSD-2 license in the LICENSE file.
################################################################################

add_executable(kagen kagen.cpp)
target_link_libraries(kagen PRIVATE kagen-core)

add_executable(interface_test interface_test.cpp)
target_link_libraries(interface_test PRIVATE kagen-core)

################################################################################
