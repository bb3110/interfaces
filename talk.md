<script type="text/javascript"
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>
# Interfacing with Python

## Olav Vahtras

Computational Python

---

layout: false

## Interfaces

### Interfacing codes

This section deals with interfaces to running codes, binary files, and compiled modules

* Automatically generated input files
* Parameter studies
* Interacting with files from external programs
* Executing Fortran or C routines

---

### Example

Use a program (here Dalton) which solves the time-independent Schrödinger equation 

$$H_0\Psi = E_0\Psi$$

and estimate the energy change due to an external electric field.


* Hamiltonian of a large part and a small perturbation

$$ H=H_0 + \epsilon z $$

$$ E=E[\epsilon] = \langle H\rangle $$

* Finite difference approximation

$$ \frac{dE}{d\epsilon} = \frac{E[\epsilon/2]+E[-\epsilon/2]}{\epsilon}$$


---

### Two input files

    **DALTON               **DALTON
    .RUN WAVE FUNCTION     .RUN WAVE FUNCTION
    **INTEGRAL             **INTEGRAL
    .DIPLEN                .DIPLEN
    *WAVE FUNCTION         *WAVE FUNCTION
    .HF                    .HF
    *HAMILTONIAN           *HAMILTONIAN
    .FIELD                 .FIELD
    .0005                  -0.0005
    ZDIPLEN                ZDIPLEN
    **END OF INPUT         **END OF INPUT

---

### Run the two calculations

```
    $ grep Final.*energy *out
    hf-_631g.out:   @ Final HF energy:              -7.897738141441                 
    hf+_631g.out:   @ Final HF energy:              -7.897779862525                 
```
Calculate the finite difference energy (cut and paste)

```
    >>> (-7.897779862525 - (-7.897738141441))/.001
    -0.04172108400002372

```

---

### The Python alternative

Set up a Python function that 

* creates input
* executes the program
* collects output
 

#### Input

Set up the input file as a triple quoted Python string.

```
>>> input = """**DALTON
... .RUN WAVE FUNCTION
... **WAVE FUNCTION
... .HF
... *HAMILTONIAN
... .FIELD
... %f
... %s
... **END OF INPUT
... """

```


---

#### Function

```
>>> import os, re

>>> def energy(strength=0, field="ZDIPLEN"):
...    inp = open('tmp.dal', 'w')
...    inp.write(input%(strength, field))
...    inp.close()
...    os.system('dalton tmp 631g')
...    out=open('tmp_631g.out', 'r')
...    for line in out:
...        if re.search('Final.*energy', line) is not None:
...            e=float(line.split(':')[1])
...            break
...    return e

```

Energy difference in Python

```
    >>> eps=0.001
    >>> print (energy(eps/2)-energy(-eps/2))/eps # doctest: +ELLIPSIS
    -0.04172108...

```

#### Sum-up

* The preceding illustrates the use of parameter studies
* Input-file is set up as a triplet quoted string with formating code
* Easy to set up input, execute and fetch output

---

### Other languages

In order to read data from a binary file written by a C- or Fortran program special tools are 

* Portability issues
* Little endian/big endian
* Compiler differences
* C/Fortran differences

### C/Fortran differences

    #include <stdio.h>          |  double precision :: pi = 3.1415927d0
    main() {                    |  open(1,'ffile',format='unformatted')
    FILE *fp;                   |  write(1) pi
    double pi = 3.1415927;      |  close(1)
    fp=fopen("cfile","wb");     |  end
    fwrite(&pi,sizeof(pi),1,fp);|
    fclose(fp);                 |
    }                           |

---

```
    $ gcc a.c && ./a.out
    $ gfortran a.f &&  ./a.out
    $ ls -l ?file
    -rw-rw-r-- 1 olav olav  8 Oct  9 07:16 cfile
    -rw-rw-r-- 1 olav olav 16 Oct  9 07:16 ffile
```

The C and Fortran program write the same data, but the resulting binary files have different size.

---

The unix command `od` (octal dump) can give information of binary files

```
    $ od -F cfile
    0000000                3.1415927
    0000010
```
--
```
    $ od -F ffile
    0000000   8.344736732028587e+127        1.7506760985e-313
    0000020
```
--

```
    $ od -D ffile
    0000000          8 1518260631 1074340347          8
    0000020
```
--
```
    $ od -F -j 4 ffile
    0000004                3.1415927                   4e-323
    0000020
```
--

#### Summary

* C binary files is a stream of bytes
* Fortran binary files are composed of records
* Each record is padded by an integer record length

---

### Binary files with Python

* `struct` module used to translate c/fortran binary data to Python objects
* `pack` function transforms Python data to c/fortran binary format
* `unpack` function transforms c/fortran binary data to Python 


### reading a Fortran record

```
>>> from struct import calcsize, unpack
>>> from numpy import array

>>> def readrec(file, dataformat='d', intsize=4):
...     datasize = calcsize(dataformat) # size of single element
...     head = file.read(intsize)       # initial integer is record size 
...     bytes = unpack('i',head)[0]     # convert to record size in bytes
...     data = file.read(bytes)         # read in a string of bytes
...     tail = file.read(intsize)       # the final integer
...     assert head == tail             # check that record sizes match
...     size = bytes/datasize           # size is number of elements
...     start = 0                       # get start/stop address
...     stop = calcsize(dataformat*size) 
...     vec = unpack(dataformat*size,data[start:stop])
...     return array(vec) # return data as numpy array

```
---

```
    >>> # open Fortran binary
    >>> f=open('ffile','rb')
    >>> arrdata=readrec(f) 
    >>> print arrdata
    [ 3.1415927]

```
    
---

#### Compiled modules

* When Python is too slow
* When algorithm prevents vectorization


#### Example ``sin(x+y)``

```
    >>> import math
    >>> def hw1(r1, r2):
    ...     return math.sin(r1+r2)
    >>> print hw1(.1,.2)
    0.295520206661

```

---

#### In Fortran

Suppose we want to evalute the function in fortran

```fortran
    # hw.f
    double precision function hw1(r1, r2)
    double precision r1, r2
    hw1  = sin(r1+r2)
    return
    end
```

We now use the `f2py` command to generate a dynamically linked library Python can import

```
    $ f2py -c -m hw hw.f > log
    $ ls -lt hw.*
    -rwxrwxr-x 1 olav olav 95954 Oct  9 08:06 hw.so
    -rw-rw-r-- 1 olav olav   255 Oct  9 08:04 hw.f
```

In Python


```
    >>> import hw
    >>> hw.hw1(.1, .2)
    0.2955202066613396

```

---

### Fortran subroutines

In fortran input and output arguments to fortran subroutines can be in any order
::

    subroutine hw2(r1, r2, s)
    double precision r1, r2, s
    s=sin(r1, r2)
    return
    end

---

The Python convention is that arguments are input and return values are output:
``s = hw2(r1,r2)``

We can supply the subroutine with information
 
```
           subroutine hw2(r1, r2, s)
           double precision r1, r2, s
    Cf2py  intent(out) s
           s=sin(r1, r2)
           return
           end
```
---

Usage: function (hw1) and subroutine (hw2) versions

```
    >>> import hw
    >>> print hw.hw1(.1,.2)
    0.295520206661
    >>> print hw.hw2(.1,.2)
    0.295520206661

```
---

### C-functions
 
```
    // hw.c
    include <math.h>
    double hw3( double r1, double r2){
    double s;
    s=sin(r1+r2);
    return s;}
```

#### Possiblities

* use f2py here as well - must write a fortran style interface
* ctypes module: compile as a shared library


Need a fortran signature file

```
    $ cat signatures.f
    real*8 function hw3(r1, r2)
    Cf2py intent(c) hw3
    real*8 r1, r2
    Cf2py intent(c) r1, r2
    end
```
---

#### With f2y:generate header file

```
    $ f2py -m hw_c -h hw.pyf signatures.f
    Reading fortran codes...
    Post-processing...
    Post-processing (stage 2)...
    Saving signatures to file "./hw.pyf"
```
---

```
    $ cat hw.pyf
    !    -*- f90 -*-
    ! Note: the context of this file is case sensitive.

    python module hw_c ! in 
        interface  ! in :hw_c
            function hw3(r1,r2) ! in :hw_c:signatures.f
                intent(c) hw3
                real*8 intent(c) :: r1
                real*8 intent(c) :: r2
                real*8 intent(c) :: hw3
            end function hw3
        end interface 
    end python module hw_c
```
---

#### Compile

```
    $ f2py -c hw.pyf hw.c > log
```
    
```
    >>> import hw_c
    >>> hw_c.hw3(.1, .2)
    0.2955202066613396

```



---

#### The ctypes module

```
    $ gcc -shared -fPIC -o hw_ctypes.so hw.c
    $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:.
```
py
```
    >>> from ctypes import CDLL, c_double
    >>> hw_lib = CDLL('hw_ctypes.so')
    >>> hw_lib.hw3.restype = c_double
    >>> s = hw_lib.hw3(c_double(.1), c_double(.2))
    >>> print s, type(s)
    0.295520206661 <type 'float'>

```

---

#### ctypes

* allows calling shared libraries
* no need to compile

```
    import ctypes 
    import ctypes.util
```

```
    #find path to libm.so
    path_to_libm = ctypes.util.find_library('m')
    #load library
    libm = ctypes.cdll.LoadLibrary(path_to_libm)
    #explain this
    libm.cos.argtypes = [ctypes.c_double]
    libm.cos.restype = ctypes.c_double

    def cos_func(arg):
        return libm.cos(arg)
```

---

* test

```
    >>> from cos_module import cos_func as cos
    >>> print cos(1.0)
    0.540302305868

```

    
---

### cython

* A Python-like language for C extensions
* A compiler 

```
    #cdef extern from "math.h":
    #    double cos(double arg)
    from libc.math cimport cos

    def cos_func(arg):
        return cos(arg)

```

---

```
    # setup.py
    from distutils.core import setup, Extension
    from Cython.Distutils import build_ext

    setup(
        cmdclass={'build_ext':build_ext},
        ext_modules=[Extension("cos_module", ["cos_module.pyx"])]
    )

```
---

* build

```
    $ python setup.py build_ext --inplace
    ...
    $ ls
    build cos_module.c cos_module.pyx cos_module.so setup.py

```

* use

```
    >>> from cos_module import cos_func as cos
    >>> cos(1.0)
    0.540302305868

```

---


#### Summary

* Input files in the form of triple quotes strings convenient for parameter studies
* The `struct` module has functions for processing binary files
* Binary files from c- and Fortran programs have differnt structure (Fortran records)
* To speed up Python code write a function in Fortran and compile it with ``f2py`` or use the ``ctypes`` module for C code
-
