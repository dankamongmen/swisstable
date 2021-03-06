# Accessing Abseil Swiss Tables from C

This is a tiny wrapper library allowing C code to use [Swiss Tables](https://abseil.io/blog/20180927-swisstables), Google's
state-of-the-art hash table implementation.

There is no pure C implementation of Swiss Tables, so they currently can't be used without a wrapper library.

I had a question: Are Swiss Tables so good that I should start using them in C code instead of [`hcreate()`](http://pubs.opengroup.org/onlinepubs/009695299/functions/hcreate.html) or even GLib's [`GHashTable`](https://developer.gnome.org/glib/stable/glib-Hash-Tables.html)?

This library is my attempt to find out.

# Building

From within the swisstable directory, do this:

```bash
$ git clone https://github.com/abseil/abseil-cpp.git
$ cmake .
$ make
```

Now you have `libswisstable.a` and `swisstable.h` that you can use from C,
remember to link against `libstdc++` and `libm`, and use `-pthread`, something
like this:

```bash
$ gcc -pthread yourcode.c libswisstable.a -lstdc++ -lm
```

To compile the benchmark use:

```bash
$ gcc -O3 -pthread -o benchmark benchmark.c libswisstable.a $(pkg-config --cflags --libs glib-2.0) -lm -lstdc++
```

The benchmark requires `libglib2.0-dev` to compare against `GHashTable`.

# Results

On my system, here are the default results:

```bash
$ ./benchmark
hash tables initialized, begin benchmark...
Trying string keys...
hsearch(3) ENTRY completed in 10 seconds, 261506 duplicates
swisstable_map_insert ENTRY completed in 3 seconds, 261506 duplicates
GHashTable ENTRY completed in 7 seconds, 261506 duplicates
Trying integer keys...
GHashTable ENTRY completed in 5 seconds, 261506 duplicates
swisstable_map_insert ENTRY completed in 3 seconds, 261506 duplicates
```

Pretty impressive default performance without any tuning. Certainly results will depend on workload, but I think you should consider switching C code to Swiss Tables if you have performance critical hash tables.

You can extend the wrapper code to allow specifying custom hashing routines, and so on (not implemented from C yet).

# Example

```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include "swisstable.h"

int main(int argc, char **argv)
{
    swisstablemap_t *swisstable = swisstable_map_create();

    if (argc != 3) {
        printf("Usage: %s key value\n", *argv);
        return 1;
    }

    swisstable_map_insert(swisstable, argv[1],          // pointer to key
                                      strlen(argv[1]),  // sizeof or strlen of key, can be an object or integer
                                      argv[2]);         // value

    char *result = swisstable_map_search(swisstable, argv[1], strlen(argv[1]));

    // For structs, integers, floats, etc you can do this
    // swisstable_map_search(swisstable, &obj, sizeof(obj));
    // This is also a swisstable_map_search_uintptr optimized for integer keys.

    printf("Key %s associated with value %s\n", argv[1], result);
    swisstable_map_free(swisstable);
    return 0;
}
```

```bash
$ gcc example.c -pthread libswisstable.a -lstdc++ -lm
```

# Notes

You can use a pointer to anything as a key - strings, structs, integers, floats, etc.

```c
swisstable_map_insert(swisstable, &key, sizeof key, value);
swisstable_map_insert(swisstable, string, strlen(string), value);
swisstable_map_insert(swisstable, buf, buflen, value);
// etc..
```

However, be aware that whatever you pass is interpreted as a buffer of size keylen, so if you're using a struct, structure padding might cause unexpected results when uninitialized data is mixed into the hash. To avoid this, either initialize structures with `calloc`/`memset`, or use `__attribute__((packed))` or equivalent to avoid padding.

You can also use `swisstable_map_create_uintptr` if you're using integer types as keys, or only want a direct comparison between pointers (The GLib equivalent would be [g_direct_equal](https://developer.gnome.org/glib/stable/glib-Hash-Tables.html#g-direct-equal)).

Currently sets and maps are exposed. A map associates keys with values, whereas a set only stores keys. You might use sets for deduplication or counting unique values, or if the key already contains the value (The GLib equivalent would be [g_hash_table_add ](https://developer.gnome.org/glib/stable/glib-Hash-Tables.html#g-hash-table-add)).

I sometimes do something like this with sets, so the value isn't necessary, I just need to lookup by key:

```c
struct foo {
 uint64_t key;
 int data1;
 int data2;
 int data3;
};

struct foo bar;

swisstable_set_insert(root, &bar, sizeof(bar.key));
```

# API

```c
// Create a new set object.
swisstableset_t * swisstable_set_create(void);

// Free a set object, note that it is your responsibility to free the keys.
void swisstable_set_free(swisstableset_t *root);

// Attempt to insert a new key into set.
// Returns key if the key was not known and successfully inserted.
// Returns existing key if already known.
void * swisstable_set_insert(swisstableset_t *root, const void *key, size_t keysize);

// Call callback on every known key.
void swisstable_set_foreach(swisstableset_t *root, void (*callback)(void *key));

// Create a new map object.
swisstablemap_t * swisstable_map_create(void);

// Free a map object, note that it is your responsibility to free the keys and values.
void swisstable_map_free(swisstablemap_t *root);

// Attempt to associate key with value.
// Returns key if the key was not known and successfully inserted.
// Returns existing key if key already known.
void * swisstable_map_insert(swisstablemap_t *root, const void *key, size_t keysize, void *value);

// Lookup a key in table.
// Returns value if key is known.
// Returns NULL if key is not known.
void * swisstable_map_search(swisstablemap_t *root, const void *key, size_t keysize);

// Call callback for every known key.
void swisstable_map_foreach(swisstablemap_t *root, void (*callback)(void *key, void *value));

// These alternatives use integers instead of pointers, so avoid some
// dereferences and overhead from creating string_view wrappers.
swisstableumap_t * swisstable_map_create_uintptr(void);
void swisstable_map_free_uintptr(swisstableumap_t *root);
void * swisstable_map_insert_uintptr(swisstableumap_t *root, uintptr_t key, void *value);
void * swisstable_map_search_uintptr(swisstableumap_t *root, uintptr_t key);
void swisstable_map_foreach_uintptr(swisstableumap_t *root, void (*callback)(uintptr_t key, void *value));

// You can give a hint about expected number of elements to avoid allocator
// overhead (RECOMMENDED).
void swisstable_set_reserve(swisstableset_t *root, size_t sizehint);
void swisstable_map_reserve(swisstablemap_t *root, size_t sizehint);
void swisstable_map_reserve_uintptr(swisstableumap_t *root, size_t sizehint);
```
