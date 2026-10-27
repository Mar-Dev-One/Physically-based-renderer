# The testbed: builds a scene in code with engine/'s PBR_Engine and draws it
# -- the engine used as a library, with no scene file and no editor.
add_executable(Testbed main.c)

target_link_libraries(Testbed PRIVATE PBR_Engine)
