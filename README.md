# Buffet

*All-inclusive Buffer for C*  
(Experiment - not yet stable or optimized)  

:orange_book: [API](#API)  

![CI](https://github.com/alcover/buffet/actions/workflows/ci.yml/badge.svg)

![schema](assets/buffet.png)  

*Buffet* is a polymorphic string buffer type featuring :
- **SSO** (small string optimization) : short data is stored inline
- **views** : no-cost references to slices of data  
- **reference counting** : secures the release of views and owned data
- **cstr** : obtain a null-terminated C string
- automated (re)allocations  

In a mere register-fitting **16 bytes**.



## How
Through a tagged union:  

```C
union Buffet {
        
    // OWN, REF, VUE
    struct {
        char*    data
        uint32_t len
        uint32_t aux:30, type:2 // aux = {cap|off}
    } ptr

    // SSO
    struct {
        char     data[15]
        uint8_t  len:6, type:2
    } sso
}
```  
The `type` tag sets how a *Buffet* is interpreted :
- `SSO` : as a char array
- `OWN` : as owning heap-allocated data
- `REF` : as a slice of owned data
- `VUE` : as a slice of other data 

The `ptr` sub-type covers :
- `OWN` : with `aux` as capacity
- `REF` : with `aux` as offset
- `VUE` : with `aux` as offset


Any *owned* data (`SSO`/`OWN`) is null-terminated.  

![schema](assets/schema.png)



## Usage

### Build & unit-test

`make && make check`

### example

```C
#include <stdio.h>
#include "buffet.h"

int main()
{
    char text[] = "The train is fast";

    Buffet vue;
    bft_strview (&vue, text+4, 5);
    bft_print(&vue);

    text[4] = 'b';
    bft_print(&vue);

    Buffet ref = bft_view (&vue, 1, 4);
    bft_print(&ref);

    char wet[] = " is wet";
    bft_append (&ref, wet, sizeof(wet));
    bft_print(&ref);

    return 0;
}
```

```
$ gcc example.c buffet -o example && ./example
train
brain
rain
rain is wet
```




# API

[bft_new](#bft_new)  
[bft_strcopy](#bft_strcopy)  
[bft_strview](#bft_strview)  
[bft_copy](#bft_copy)  
[bft_view](#bft_view)  
[bft_append](#bft_append)  
[bft_free](#bft_free)  

[bft_cap](#bft_cap)  
[bft_len](#bft_len)  
[bft_data](#bft_data)  
[bft_cstr](#bft_cstr)  
[bft_export](#bft_export)  

[bft_print](#bft_print)  
[bft_dbg](#bft_dbg)  


### bft_new
```C
void bft_new (Buffet *dst, size_t cap)
```
Create a *Buffet* of capacity at least `cap`.  

```C
Buffet buf;
bft_new(&buf, 20);
bft_dbg(&buf); 
// type:OWN cap:32 len:0 data:''
```

### bft_strcopy
```C
void bft_strcopy (Buffet *dst, const char *src, size_t len)
```
Copy `len` bytes from `src` into new `dst`.  

```C
Buffet copy;
bft_strcopy(&copy, "Bonjour", 3);
bft_dbg(&copy); 
// type:SSO cap:14 len:3 data:'Bon'

```

### bft_strview
```C
void bft_strview (Buffet *dst, const char *src, size_t len)
```
View `len` bytes from `src` into new `dst`.  
You get a window into `src`. No copy or allocation is done.

```C
char src[] = "Eat Buffet!";
Buffet view;
bft_strview(&view, src+4, 3);
bft_dbg(&view);
// type:VUE cap:0 len:6 data:'Buffet!'
bft_print(&view);
// Buf
```

### bft_copy
```C
Buffet bft_copy (const Buffet *src, ptrdiff_t off, size_t len)
```
Create new *Buffet* by copying `len` bytes from [data(`src`) + `off`].  
The return is an independant owning Buffet.


### bft_view
```C
Buffet bft_view (const Buffet *src, ptrdiff_t off, size_t len)
```
Create new *Buffet* by viewing `len` bytes from [data(`src`) + `off`].  
The return is internally either 
- a reference to `src` if `src` is owning
- a reference to `src`'s origin if `src` is itself a REF
- a view on `src` data if `src` is SSO or VUE

`src` now cannot be released before either  
- the return is released
- the return is detached as owner (e.g. when you `append()` to it).

```C
Buffet src;
bft_strcopy(&src, "Bonjour", 7);
Buffet ref = bft_view(&src, 0, 3);
bft_dbg(&ref);   // type:VUE cap:0 len:6 data:'Buffet!'
bft_print(&ref); // Bon
```


### bft_free
```C
void bft_free (Buffet *buf)
```
If *buf* is SSO or VUE, it is simply zeroed, making it an empty SSO.  
If *buf* is REF, the refcount is decremented and *buf* zeroed.  
If *buf* owns data :  
- with no references, the data is released and *buf* is zeroed
- with live references, *buf* is marked for release and waits for its last ref to be released.  


```C
char text[] = "Le grand orchestre de Patato Valdez";

Buffet own;
bft_strcopy(&own, text, sizeof(text));
bft_dbg(&own);
// type:OWN data:'Le grand orchestre de Patato Valdez'

Buffet ref = bft_view(&own, 22, 13);
bft_dbg(&ref);
// type:REF data:'Patato Valdez'

// Too soon but marked for release
bft_free(&own);
bft_dbg(&own);
// type:OWN data:'Le grand orchestre de Patato Valdez'

// Release last ref, hence owner
bft_free(&ref);
bft_dbg(&own);
// type:SSO data:''
```



### bft_append
```C
size_t bft_append (Buffet *dst, const char *src, size_t len)
```
Appends `len` bytes from `src` to `dst`.  
Returns new length or 0 on error.
If over capacity, `dst` gets reallocated. 

```C
Buffet buf;
bft_strcopy(&buf, "abc", 3); 
size_t newlen = bft_append(&buf, "def", 3); // newlen == 6 
bft_dbg(&buf);
// type:SSO cap:14 len:6 data:'abcdef'
```




### bft_cap  
Get current capacity.  
```C
size_t bft_cap (Buffet *buf)
```

### bft_len  
Get current length.  
```C
size_t bft_len (Buffet *buf)`
```

### bft_data
Get current data pointer.  
```C
const char* bft_data (const Buffet *buf)`
```
If *buf* is a ref/view, `strlen(bft_data)` may be longer than `buf.len`. 

### bft_cstr
Get current data as a NUL-terminated **C string**.  
```C
const char* bft_cstr (const Buffet *buf, bool *mustfree)
```
If REF/VUE, the slice is copied into a fresh C string that must be freed.

### bft_export
 Copies data up to `buf.len` into a fresh C string that must be freed.
```C
char* bft_export (const Buffet *buf)
```



### bft_print
Prints data up to `buf.len`.
```C
void bft_print (const Buffet *buf)`
```

### bft_dbg  
Prints *buf* state.  
```C
void bft_dbg (Buffet *buf)
```

```C
Buffet buf;
bft_strcopy(&buf, "foo", 3);
bft_dbg(&buf);
// type:SSO cap:14 len:3 data:'foo'
```