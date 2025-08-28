
On x86-64, the user space enters the kernel via the syscall instruction. The kernel's ``sys_call_table`` is an indexed array of function pointers. The index = syscall number (``__NR_*``). Each entry points to a C function with the syscall ABI (for example ``asmlinkage long __x64_sys_openat(...)`` or equivalently ``sys_openat`` in some trees).

What is the point in hooking it? 
- Kprobes observe the function that you attach it to
- A table hook interposes; you are able to see every call to that service and change the service's functionality (by overwriting the function pointer in the table).

