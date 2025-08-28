
To hook the syscall table, you will need its address. 
- Sometimes ``kallsyms_lookup_name()`` is exported, but sometimes it isn't. 

You are able to attach a kprobe to ``kallsyms_lookup_name()`` to be able to retrieve the address of the syscall table.

Fundamentally, to resolve symbols in the kernel, you need two things: the function ``kallsyms_lookup_name`` (to preform arbitrary symbol lookups), and the data object ``sys_call_table``.

There are some limitations. For example: 
- `kallsyms_lookup_name` is **not always exported** to modules.
- ``kprobes`` can resolve function symbols, not data
- ``sys_call_table`` is data, so Kprobes can't grab it directly
So we do a bootstrap:

Step 1: Acquire ``kallsmys``: 

```c
static unsigned long lookup_symbol(const char *name) {
	struct kprobe kb = { .symbol_name = name }; 
	// temp kprobe struct 
	usinged long addr = 0;
	if (register_kprobe(&kp) == 0) {
		addr = (unsigned long)kp.addr; //resolved address
		unregister_kprobe(&kp);	
	}
	return addr;
}
```
- This function returns the address of a given symbol (``name``) and returns its address (or 0 if not found).
- We also don't even need to fire the probe, simply registering it will give us the address in ``kp.addr``


Step 2 - Use the lookup to find ``sys_call_table``:
Then, in ``module_init``:
```c
typedef unsinged long (*kln_t)(const char *);
static kln_t kln;
static void **sys_call_table;

usigned long kln_addr = lookup_symbol("``kallsyms_lookup_name");
if (!kln_addr) 
	return -ENOENT;
kln = (kln_t)kln_addr;

sys_call_table = (void **)kln("sys_call_table");
if (!sys_call_table)
	return -ENOENT;
```

Break down of what this does:

```c 
typedef unsinged long (*kln_t)(const char *);
```
- Defines a function pointer type named ``kln_t``
- A pointer to a function taking ``const char *`` and returning an ``unsigned long`` 
- This is a recreation of the real kernel function:
```c
`unsigned long kallsyms_lookup_name(const char *name);
```

```c
static kln_t kln;
```
- This variable will hold the address of ``kallsyms_lookup_name`` once we find it.

```c
static void **sys_call_table; 
```
- This will act as the address of the syscall table
- The syscall table can be treated like an array of void pointers and then cast into the appropriate type of function pointer 

```c
`unsigned long kln_addr = lookup_symbol("kallsyms_lookup_name");`
```
- Calls the created helper that was created above

```c
if (!kln_addr) return -ENOENT;
```
- Returns "No such entry" if the address is 0
- What this error means 
	- ``CONFIG_KPROBES`` is off 
	- The symbol doesn't exist on this kernel
	- you're on a configuration that blocks kprobe resolution of that function 

```c
kln = (kln_t)kln_addr;
```
- Type case the address into the function pointer type that we created before. This type matches that of the ``kallsyms_lookup_name(const char *name)``

```c
sys_call_table = (void **)kln("sys_call_table");`
```
- Call the real ``kalsyms_lookup_name`` (via the ``kln`` function pointer that we previously created) to get the address of the data symbol ``"sys_call_table"``, then cast it into a ``void **`` 
- ``kallsyms_lookup_name`` returns an address (or unsigned long) for any symbol (function or data). 
	- For data symbols, you cast to the appropriate pointer type (in our case its a pointer to pointers)

```c
if (!sys_call_table) return -ENOENT;
```
- if lookup fails, return "No such entry"
- this could be returned if ``sys_call_table`` is hidden or renamed. 

Now we have the address of sys_call_table