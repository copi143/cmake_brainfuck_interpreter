# vimtcompile:[ -d build ] && rm -rf build; cmake -B build
cmake_minimum_required(VERSION 3.31)
project(BrainfuckInterpreter)

set(INPUT_FILE
    test.bf
    CACHE FILEPATH "Input file path")
set(STDIN_FILE
    program_in
    CACHE FILEPATH "Path of standard input for the program")

file(READ "${INPUT_FILE}" SRC)
file(READ "${STDIN_FILE}" STDIN)
string(LENGTH "${STDIN}" STDIN_LEN)
set(STDIN_PTR 0)

set(STDOUT_BUF "")

set(ascii_string " ")
foreach(i RANGE 1 255)
  string(ASCII ${i} i)
  string(APPEND ascii_string ${i})
endforeach()

# --------------------------------------------------
# %             Compile Brainfuck Code             %
# --------------------------------------------------

set(instruction_list) # Output instruction sequence
set(data_list) # Output data sequence

set(value_add_count 0)
set(pointer_move_count 0)
set(jump_point_list) # Jump point list (positions of left brackets)
set(IN_COMMENT OFF)

function(insert_value_add)
  if(NOT value_add_count EQUAL 0)
    list(APPEND instruction_list "+")
    math(EXPR value_add_count "${value_add_count} % 256")
    list(APPEND data_list ${value_add_count})
    set(value_add_count
        0
        PARENT_SCOPE)
    set(instruction_list
        ${instruction_list}
        PARENT_SCOPE)
    set(data_list
        ${data_list}
        PARENT_SCOPE)
  endif()
endfunction(insert_value_add)

function(insert_pointer_move)
  if(NOT pointer_move_count EQUAL 0)
    list(APPEND instruction_list ">")
    list(APPEND data_list ${pointer_move_count})
    set(pointer_move_count
        0
        PARENT_SCOPE)
    set(instruction_list
        ${instruction_list}
        PARENT_SCOPE)
    set(data_list
        ${data_list}
        PARENT_SCOPE)
  endif()
endfunction(insert_pointer_move)

string(LENGTH "${SRC}" SRC_LEN)
foreach(index RANGE 0 ${SRC_LEN})
  string(SUBSTRING ${SRC} ${index} 1 ch)
  if(ch STREQUAL "\n")
    set(IN_COMMENT OFF)
    continue()
  endif()
  if(ch STREQUAL "#")
    set(IN_COMMENT ON)
    continue()
  endif()
  if(IN_COMMENT)
    continue()
  endif()
  if(ch STREQUAL "+")
    insert_pointer_move()
    math(EXPR value_add_count "${value_add_count} + 1")
  endif()
  if(ch STREQUAL "-")
    insert_pointer_move()
    math(EXPR value_add_count "${value_add_count} + 255") # We don't want negative numbers in the data, so we add 255 instead of subtracting 1
  endif()
  if(ch STREQUAL "<")
    insert_value_add()
    math(EXPR pointer_move_count "${pointer_move_count} - 1")
  endif()
  if(ch STREQUAL ">")
    insert_value_add()
    math(EXPR pointer_move_count "${pointer_move_count} + 1")
  endif()
  if(ch STREQUAL "[")
    insert_pointer_move()
    insert_value_add()
    list(LENGTH jump_point_list jump_point_list_len)
    list(LENGTH instruction_list instruction_len)
    math(EXPR target "${instruction_len} + ${jump_point_list_len}")
    list(APPEND jump_point_list ${target})
  endif()
  if(ch STREQUAL "]")
    insert_pointer_move()
    insert_value_add()
    list(LENGTH jump_point_list jump_point_list_len)
    list(LENGTH instruction_list instruction_len)
    if(jump_point_list_len EQUAL 0)
      message(FATAL_ERROR "Unbalanced brackets found in source")
    endif()
    math(EXPR target "${instruction_len} + ${jump_point_list_len}")
    list(POP_BACK jump_point_list left)
    math(EXPR pos "${left} - ${jump_point_list_len} + 1")
    list(INSERT instruction_list ${pos} "l")
    list(INSERT data_list ${pos} ${target})
    math(EXPR target "${left}")
    list(APPEND instruction_list "r")
    list(APPEND data_list ${target})
  endif()
  if(ch STREQUAL ".")
    insert_pointer_move()
    insert_value_add()
    list(APPEND instruction_list ".")
    list(APPEND data_list 0)
  endif()
  if(ch STREQUAL ",")
    insert_pointer_move()
    insert_value_add()
    list(APPEND instruction_list ",")
    list(APPEND data_list 0)
  endif()
endforeach()

list(LENGTH jump_point_list jump_point_list_len)
if(jump_point_list_len GREATER 0)
  message(FATAL_ERROR "Unbalanced brackets found in source")
endif()

# --------------------------------------------------
# %               Run Brainfuck Code               %
# --------------------------------------------------

set(index 0)
set(pointer 0)

list(LENGTH instruction_list instruction_len)
while(${index} LESS ${instruction_len})
  list(GET instruction_list ${index} ch)
  list(GET data_list ${index} data)
  if(ch STREQUAL "+")
    math(EXPR "RAM_${pointer}" "(${RAM_${pointer}} + ${data}) % 256")
  elseif(ch STREQUAL ">")
    math(EXPR pointer "${pointer} + ${data}")
  elseif(ch STREQUAL "l")
    math(EXPR ret "${RAM_${pointer}} + 0")
    if("${ret}" EQUAL 0)
      set(index ${data})
    endif()
  elseif(ch STREQUAL "r")
    math(EXPR ret "${RAM_${pointer}} + 0")
    if(NOT "${ret}" EQUAL 0)
      set(index ${data})
    endif()
  elseif(ch STREQUAL ".")
    math(EXPR ret "${RAM_${pointer}} + 0")
    string(ASCII ${ret} ret)
    if(ret STREQUAL "\n")
      message("${STDOUT_BUF}")
      set(STDOUT_BUF "")
    else()
      set(STDOUT_BUF "${STDOUT_BUF}${ret}")
    endif()
  elseif(ch STREQUAL ",")
    if(STDIN_PTR LESS STDIN_LEN)
      string(SUBSTRING "${STDIN}" ${STDIN_PTR} 1 value)
      string(FIND "${ascii_string}" "${value}" value)
      math(EXPR STDIN_PTR "${STDIN_PTR} + 1")
    else()
      set(${value} 0)
    endif()
    set("RAM_${pointer}" ${value})
  endif()
  math(EXPR index "${index} + 1")
endwhile()

if(NOT STDOUT_BUF STREQUAL "")
  message("${STDOUT_BUF}")
endif()
