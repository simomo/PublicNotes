# How to compile and run cmocka test cases on Windows

## Folder structure

```
project_root
  +
  |
  +-+module_a
       +
       +-+impl_smth.c
       |
       +-+impl_smth.h
       |
       +-+tests
            +
            +-+test_impl_smth.c
```

## Compiling command

```shell
gcc -I"D:\Program Files (x86)\cmocka\include" -I"C:/MinGW/include/*" -L"D:\Program Files (x86)\cmocka\lib" -L"C:\MinGW\lib" ../cmd_store.c cmd_store_test.c -l libcmocka
```

## Explanation

### Reference

> [1] [GCC and Make - Compiling, Linking and Building C/C++ Applications](https://www3.ntu.edu.sg/home/ehchua/programming/cpp/gcc_make.html)

### `-I`

Make gcc can find standard header files(`stddef.h` .etc) and cmocka's header file (e.g. `cmocka.h`)

### `-L`

Make gcc can ***find*** standard libs and cmocka's lib file(e.g. `libcmocka.dll.a`)

### `-l`

Make gcc link cmocka's lib file while compiling.
And, the order matters, we must set `-l` after `cmd_store_test.c`.

### `../cmd_store.c`

Make gcc compile `cmd_store.c` and link it to our test cases.