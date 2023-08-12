---
title: Abusing COW to create system wide and process specific patches
layout: post
---

![crazy-cow](/assets/crazy-cow.png)

# Introduction

Imagine if a program needed to load a library that has already been loaded by other programs in the system.
The operating system would have to load once again that same library just for that program, which would result in having multiple copies of it in memory.

Apply this concept to a library that's loaded by all programs in the system, for example ntdll on Windows systems, you can understand that having all of 
these copies of the library is a huge waste of memory.

![with-cow](/assets/wo-cow.png)

This is what copy on write solves, it allows multiple programs to share the same memory until one of them modifies it.

![with-cow](/assets/w-cow.png)

To achieve this, the memory pages are marked with the copy on write protection in the page tables so that the processor generates a page fault if it tries to modify them.

The page fault would then be handled by the operating system which allocates a new copy of the page and maps it into the program that's trying to modify it.
This ensures that the changes made by the program don't relfect in the other programs as well.

# Getting around Copy On Write

Getting around this means that we need to be able to modify the memory without the processor generating a page fault, so that the remapping process is never started to begin with.

The processor generates a page fault because during the virtual to physical memory translation it finds out that the memory is marked with the copy on write protection. Getting past this check is pretty trivial as we can just map the physical memory page in our address space without the copy on write protection which would allow us to have our changes reflected in all of the programs.

# Getting physical memory access

A common way of getting physical memory access is through the use of a third party signed driver that exposes some functionality that we can use for the managing of physical memory from our address space.

A driver that we can use is Intel's driver iqvw64e.sys, it exposes functions for the mapping and unmapping of physical memory, which are pretty much just wrappers around MmMapIoSpace and MmUnmapIoSpace. It also exposes a function for the translation of virtual addresses to physical addresses, which is also just a wrapper around MmGetPhysicalAddress.

The driver uses ioctl as the communication method and divides its ioctl request handlers in four categories each one identified by its ioctl control code. The request is then dispatched by the category request dispatcher and fed to the request handler identified by the operation_id field in the input buffer.

```cpp
// ioctl control code that identifies the memory category
static constexpr auto memory_category = 0x80862007;

// identifies a request handler in the category
enum operation : int64_t {
  virtual_to_physical = 0x25,
  map_physical = 0x39,
  unmap_physical = 0x3A
};

// the request sent to the driver
struct request {
  operation operation_id;   // 0x0
  int64_t reserved;         // 0x8
  void* result;             // 0x10
  void* virtual_address;    // 0x18
  void* physical_address;   // 0x20
  size_t number_of_bytes;   // 0x28
  bool is_user_mapping;     // 0x30
};
```

Now all we need to do is just build a request with the desired operation and send it to the driver with the control code identifying the memory category, the driver will take care of everything else.

```cpp
class request_builder {
 public:
  // builds a request for the translation of virtual addresses in physical ones
  request virtual_to_physical(void* address);

  // builds a request for the mapping of physical memory
  request map_physical(void* address, size_t size);

  // builds a request for the unmapping of physical memory
  request unmap_physical(void* address, size_t size);
};

class iqvw64e {
 public:
  // opens a handle to the device driver using CreateFile
  explicit iqvw64e(const std::string& device_name);

  // closes the handle to the device driver using CloseHandle
  ~iqvw64e();

  // builds and sends a virtual to physical request
  void* get_physical(void* address);

  // builds and sends a physical memory mapping request
  void* map_physical(void* address, size_t size);

  // builds and sends a physical memory unmapping request
  bool unmap_physical(void* address, size_t size);
  
  // sends the request (DeviceIoControl wrapper)
  bool send_request(request& request);

  // checks if the handle is valid
  bool is_initialized();

 private:
  request_builder builder_;
  HANDLE device_handle_;
  bool is_initialized_;
};
```

# Copy On Write protected binaries

In Windows for a program to load a library utilizing the copy on write technique, it needs to be able to allocate it at the same address that the other programs have it loaded at, in any other case a new copy is allocated. 
With ASLR-enabled binaries this is almost always possible as each one will have it's own randomized unique address, making it less likely to be reallocated.

This concept applies for both executables and libraries, allowing us to create process specific and system wide patches, by either patching executables or libraries. The process is the same in both cases, we load an arbitrary target binary and patch its physical memory, the patches will reflect and persist until the system is restarted.

# Address space limitations

Someone might be tempted to load a kernel driver and patch it thinking that the patches would reflect in kernel space however, as we previously discussed, this is not possible.

The user space program would never be able to allocate memory at the address that the driver is loaded at as it's outside of its address space, while if we were to load it first in our program and then in the kernel, it would just get reallocated because it's outside of the kernels address space.

# Patching a third party library system wide

We can also choose to patch a third party library to target a specific family of programs, to understand how the whole thing works I've written a test library that exports a "get_number" function, the function just returns an integer of 5555 which we will patch so it returns 1337. 

The patches we apply, as previously mentioned, will reflect in all the programs that have the library currently loaded as well as in the ones that will load it in the future.

```cpp
// loading the library
auto library = LoadLibraryA("nekka.dll");

// getting the virtual address of the get_number function
auto get_number = reinterpret_cast<int(*)()>(
  GetProcAddress(library, "get_number"));

// performing the virtual to physical translation
auto physical_addr = intel_driver.get_physical(get_number);

// mapping a page of the physical memory in our address space
auto mapping = intel_driver.map_physical(physical_addr, 0x1000);

std::array<uint8_t, 6> shellcode {
  0xB8, 0x39, 0x05, 0x00, 0x00,  // mov eax, 1337
  0xC3                           // ret
};

// writing our shellcode
memcpy(mapping, shellcode.data(), shellcode.size());

// unmapping the page we previously mapped
intel_driver.unmap_physical(mapping, 0x1000);

// freeing the library
FreeLibrary(library);
```

![poc](/assets/poc.png)

As we can see it works, we load the library, map a page of it's physical memory in our address space, write the shellcode and free it, then when a program tries to load it the patches are still in place.

# Credits
- mrexodia: thanks for the advices and helping me structure this :D