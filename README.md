# Brainfuck Interpreter in CMake

This is a Brainfuck interpreter written in CMake scripting language.

To run the interpreter, simply type

```shell
cmake -B build
```

You can also use these command-line options to customize input file paths:

```shell
# Path to the source file, default: test.bf
-DINPUT_FILE=path/to/your/source/file
# Path to input file for the program, default: program_in
-DSTDIN_FILE=path/to/your/stdin/file
```
