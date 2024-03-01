cmake_minimum_required(VERSION 3.13)

add_library(dhara INTERFACE)

target_sources(dhara INTERFACE
    dhara/error.c
    dhara/journal.c
    dhara/map.c
)

target_include_directories(dhara INTERFACE .)
