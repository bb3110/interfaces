<!-- 
>>> tmpdir = getfixture('tmpdir')

-->

# Computational Python

<script type="text/javascript"
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>

## Interfacing with Python

### Olav Vahtras

---

layout: false

## Interfacing with Python


### Interfacing with other codes

<font color="red">
* Text files
</font>
  - Automatic generation of input files (parameter studies)
  - Getting data from output file
* Binary files
  - Getting data from binary files written by C and Fortran programs

### Interfacing with other languages

* Calling Fortran from Python
* Calling C from Python

---

## Interfacing with other codes

### Text files: a manual example

Use a program (here Dalton) which solves the time-independent Schrödinger equation 

$$H_0\Psi = E_0\Psi$$

and estimate the energy change due to an external electric field.


* The Hamiltonian has a large part and a small perturbation

$$ H=H_0 + \epsilon z $$

$$ E=E[\epsilon] = \langle H\rangle $$

* Calculate the change using the finite difference approximation

$$ \frac{dE}{d\epsilon} = \frac{E[\epsilon/2]+E[-\epsilon/2]}{\epsilon}$$


---

* Two input files

```
    **DALTON                         **DALTON
    .RUN WAVE FUNCTION               .RUN WAVE FUNCTION
    **INTEGRAL                       **INTEGRAL
    .DIPLEN                          .DIPLEN
    *WAVE FUNCTION                   *WAVE FUNCTION
    .HF                              .HF
    *HAMILTONIAN                     *HAMILTONIAN
    .FIELD                           .FIELD
    .0005                            -0.0005
    ZDIPLEN                          ZDIPLEN
    **END OF INPUT                   **END OF INPUT
```

--

* Run the two calculations

```
    $ grep Final.*energy *out
    hf-_631g.out:   @ Final HF energy:              -7.897738141441                 
    hf+_631g.out:   @ Final HF energy:              -7.897779862525                 
```
--
* Calculate the finite difference energy (cut and paste)

```
    >>> (-7.897779862525 - (-7.897738141441))/.001
    -0.04172108400002372

```

---

### Text files: same example automated

Set up a Python function that 

* creates input
* executes the program
* collects output
 
---

* Parameterized input file

Set up the input file as a triple quoted Python string.

```
>>> dalton_input = """**DALTON
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

<!--
>>> molecule_input = """BASIS 
... 6-31G
... First title line --------------------------------------------------------------
... Second title line--------------------------------------------------------------
...    2    0
...        3.    1   
... A   0.0  0.0  -1.0
...        1.    1 
... B   0.0  0.0   1.0
... """

-->

* The `%s` (string) and `%f` (float) are string control characters familiar for C programmers
* This is a template that can generate input files for different fields types and strengths

```
>>> print(dalton_input % (1.0, "foobar"))
**DALTON
.RUN WAVE FUNCTION
**WAVE FUNCTION
.HF
*HAMILTONIAN
.FIELD
1.000000
foobar
**END OF INPUT

```

---

* A single function to do it all

```
>>> import re, subprocess, tempfile

>>> def energy(strength=0, field="ZDIPLEN"):
...    #write input
...    with open("%s/_%s-%s.dal" % (tmpdir, strength, field), 'w') as dalinp:
...        dalinp.write(dalton_input % (strength, field))
...    with open("%s/%s.mol" % (tmpdir, "tmp"), 'w') as inp:
...        inp.write(molecule_input)
...    # run program
...    subprocess.call(
...       "chdir %s && dalton _%s-%s tmp" % (tmpdir, strength, field),
...       stdout=tempfile.TemporaryFile(),
...       stderr=tempfile.TemporaryFile(),
...       shell=True
...       )
...    # fetch output
...    with open("%s/_%s-%s_tmp.out" % (tmpdir, strength, field), 'r') as dalout:
...       for line in dalout:
...           if re.search('Final.*energy', line) is not None:
...               e=float(line.split(':')[1])
...               break
...    return e

```

--

Energy difference in Python

```
    >>> eps=0.001
    >>> print((energy(eps/2)-energy(-eps/2))/eps)                                 # doctest: +ELLIPSIS
    -0.04172108...

```

---
Better still: split tasks to separate functions

*   A write function

```
>>> def write_input(strength, field):
...    with open("%s/_%s-%s.dal" % (tmpdir, strength, field), 'w') as dalinp:
...        dalinp.write(dalton_input%(strength, field))
...    with open("%s/%s.mol" % (tmpdir, "tmp"), 'w') as molinp:
...        molinp.write(molecule_input)

```

*  A run function

```
>>> def  run_program(strength, field):
...    subprocess.call(
...       "chdir %s && dalton _%s-%s tmp" % (tmpdir, strength, field),
...       shell=True
...       )

```

*   A read function

```
>>> def  fetch_output(strength, field):
...    with open("%s/_%s-%s_tmp.out" % (tmpdir, strength, field), 'r') as dalout:
...       for line in dalout:
...           if re.search('Final.*energy', line) is not None:
...               e=float(line.split(':')[1])
...               break
...    return e

```

---

* Redefined `energy`


```
>>> def energy(strength=0.0, field="ZDIPLEN"):                                    #doctest: +ELLIPSIS
...     write_input(strength, field)
...     run_program(strength, field)
...     return fetch_output(strength, field)

```

```
    >>> eps=0.001
    >>> print((energy(eps/2)-energy(-eps/2))/eps)                                 #doctest: +ELLIPSIS
    -0.04172108...

```

* Next step gradient function

```
>>> def gradient(f, eps=0.001):
...     return (f(eps/2) - f(-eps/2))/eps

```

```
>>> gradient(energy)                                                              #doctest: +ELLIPSIS
-0.04172108...

```

---

### Summary

* The preceding examples illustrates the use of parameter studies
* Input-file is set up as a triplet quoted string with formating code
* Easy to set up input, execute and fetch output

---

### Binary files

In order to read data from a binary file written by a C- or Fortran program special tools are 

* Portability issues
* Little endian/big endian
* Compiler differences
* C/Fortran differences

---

### C/Fortran differences

    # a.c                       |  ! a.f90
    #include <stdio.h>          |  double precision :: pi = 3.1415927d0
    main() {                    |  open(1,'ffile',format='unformatted')
    FILE *fp;                   |  write(1) pi
    double pi = 3.1415927;      |  close(1)
    fp=fopen("cfile","wb");     |  end
    fwrite(&pi,sizeof(pi),1,fp);|
    fclose(fp);                 |
    }                           |

<!--
>>> with open("a.c", "w") as csource:
...     csource.write("""\
... #include <stdio.h>
... void main() {                    
... FILE *fp;                   
... double pi = 3.1415927;      
... fp=fopen("cfile","wb");     
... fwrite(&pi,sizeof(pi),1,fp);
... fclose(fp);                 
... }"""
...     )
199
>>> with open("a.f90", "w") as fsource:
...    fsource.write("""\
... double precision :: pi = 3.1415927d0
... open(1,file='ffile',form='unformatted')
... write(1) pi
... close(1)
... end
... """
... )
102
>>> import subprocess
>>> _ = subprocess.call('gcc a.c -o ac.x && ./ac.x && rm a.c', shell=True)
>>> _ = subprocess.call('gfortran a.f90 -o af.x && ./af.x && rm a.f90', shell=True)

-->

    $ gcc a.c && ./a.out
    $ gfortran a.f90 &&  ./a.out
    $ ls -l ?file
    -rw-rw-r-- 1 olav olav  8 Oct  9 07:16 cfile
    -rw-rw-r-- 1 olav olav 16 Oct  9 07:16 ffile


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

### Summary

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
...     size = bytes//datasize          # size is number of elements
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
    >>> print(arrdata)
    [ 3.1415927]

```
    
---

## Compiled modules: Fortran and C

* When Python is too slow
* When algorithm prevents vectorization


* Example ``sin(x+y)``

```
    >>> import math
    >>> def hw1(r1, r2):
    ...     return math.sin(r1+r2)
    >>> print(hw1(.1,.2))
    0.2955202066613396

```

---

### Fortran functions

Suppose we want to evalute the function in fortran

```fortran
    # hw.f
    double precision function hw1(r1, r2)
    double precision r1, r2
    hw1  = sin(r1+r2)
    return
    end
```
<!--
>>> with open('hw.f90', 'w') as hwf:
...     hwf.write("""\
... # hw.f90
... double precision function hw1(r1, r2)
... double precision r1, r2
... hw1  = sin(r1+r2)
... return
... end
...
... subroutine hw2(r1, r2, s)
... double precision r1, r2, s
... !f2py  intent(out) s
... s = sin(r1+r2)
... return
... end
... """)
201

-->

We now use the `f2py` command to generate a dynamically linked library Python can import

```
    $ f2py -c -m hw hw.f > log
    $ ls -lt hw.*
    -rwxrwxr-x 1 olav olav 95954 Oct  9 08:06 hw.so
    -rw-rw-r-- 1 olav olav   255 Oct  9 08:04 hw.f
```

<!--
>>> _ = subprocess.call('f2py -c -m hw hw.f90', shell=True)

-->

In Python


```
>>> import hw
>>> hw.hw1(.1, .2)
0.2955202066613396

```

---

### Fortran subroutines

In fortran input and output arguments to fortran subroutines can be in any order

```
subroutine hw2(r1, r2, s)
double precision r1, r2, s
s=sin(r1+r2)
return
end
```

--

The Python convention is that arguments are input and return values are output:
``s = hw2(r1,r2)`` We can supply the subroutine with information
 
```
subroutine hw2(r1, r2, s)
double precision r1, r2, s
!f2py  intent(out) s
s=sin(r1+r2)
return
end
```

Usage: function (hw1) and subroutine (hw2) versions

```
    >>> import hw
    >>> print(hw.hw1(.1,.2))
    0.2955202066613396
    >>> print(hw.hw2(.1,.2))
    0.2955202066613396

```
---

### C-functions
 
```
    // hw.c
    include <math.h>
    double hw3(double r1, double r2){
        double s;
        s=sin(r1+r2);
        return s;
        }
```
<!--
>>> with open('hw.c', 'w') as hwc:
...     n = hwc.write("""\
... // hw.c
... #include <math.h>
... double hw3(double r1, double r2){
...     double s;
...     s=sin(r1+r2);
...     return s;
... }
... """)

-->

Possiblities

* use f2py here as well - must write a fortran style interface
* ctypes module: compile as a shared library


Need a fortran signature file

```
    #signatures.f90
    real*8 function hw3(r1, r2)
    !f2py intent(c) hw3
    real*8 r1, r2
    !f2py intent(c) r1, r2
    end
```
<!--
>>> with open('signatures.f90', 'w') as sigf:
...     n = sigf.write("""\
... !signatures.f90
... real*8 function hw3(r1, r2)
... !f2py intent(c) hw3
... real*8 r1, r2
... !f2py intent(c) r1, r2
... end
... """)

-->
---

* With f2y:generate header file

```
    $ f2py -m hw_c -h hw.pyf signatures.f90
    Reading fortran codes...
    Post-processing...
    Post-processing (stage 2)...
    Saving signatures to file "./hw.pyf"
```
<!--
>>> _ = subprocess.call('f2py -m hw_c -h hw.pyf signatures.f90', shell=True)

-->

--

```
    $ cat hw.pyf
    !    -*- f90 -*-
    ! Note: the context of this file is case sensitive.

    python module hw_c ! in 
        interface  ! in :hw_c
            function hw3(r1,r2) ! in :hw_c:signatures.f90
                intent(c) hw3
                real*8 intent(c) :: r1
                real*8 intent(c) :: r2
                real*8 intent(c) :: hw3
            end function hw3
        end interface 
    end python module hw_c
```
---

* Compile

```
    $ f2py -c hw.pyf hw.c > log
```
<!--
>>> _ = subprocess.call('f2py -c hw.pyf hw.c', shell=True)

-->
    
```
    >>> import hw_c
    >>> hw_c.hw3(.1, .2)
    0.2955202066613396

```



---

### The ctypes module

```
    $ gcc -shared -fPIC -o hw_ctypes.so hw.c
    $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:.
```

<!--
>>> _ = subprocess.call('gcc -shared -fPIC -o hw_ctypes.so hw.c', shell=True)

-->

```
>>> from ctypes import CDLL, c_double
>>> hw_lib = CDLL('hw_ctypes.so')
>>> hw_lib.hw3.restype = c_double
>>> s = hw_lib.hw3(c_double(.1), c_double(.2))
>>> print(s, type(s))
0.2955202066613396 <class 'float'>

```

---

### ctypes

* allows calling shared libraries
* no need to compile

```
>>> import ctypes 
>>> import ctypes.util

```

```
>>> #find path to libm.so
>>> path_to_libm = ctypes.util.find_library('m')
>>> #load library
>>> libm = ctypes.cdll.LoadLibrary(path_to_libm)
>>> libm.cos.argtypes = [ctypes.c_double]
>>> libm.cos.restype = ctypes.c_double
>>> def cos_func(arg):
...     return libm.cos(arg)

```

--

* test

```
>>> print(cos_func(1.0))
0.5403023058681398

```

    
---

### cython

* A Python-like language for C extensions, a compiler 

```
    // cos_module.pyx
    #cdef extern from "math.h":
    #    double cos(double arg)
    from libc.math cimport cos

    def cos_func(arg):
        return cos(arg)

```
<!--
>>> with open('cos_module.pyx', 'w') as pyx:
...     _ = pyx.write("""\
... #cdef extern from "math.h":
... #    double cos(double arg)
... from libc.math cimport cos
...
... def cos_func(arg):
...     return cos(arg)
... """)

-->

--

```
    # setup.py
    from distutils.core import setup, Extension
    from Cython.Distutils import build_ext

    setup(
        cmdclass={'build_ext':build_ext},
        ext_modules=[Extension("cos_module", ["cos_module.pyx"])]
    )

```
<!--
>>> with open('setup.py', 'w') as setuppy:
...    _ = setuppy.write("""\
... # setup.py
... from distutils.core import setup, Extension
... from Cython.Distutils import build_ext
...
... setup(
...     cmdclass={'build_ext':build_ext},
...     ext_modules=[Extension("cos_module", ["cos_module.pyx"])]
... )
... """)

-->

--

* build and use

```
    $ python setup.py build_ext --inplace

```
<!--
>>> _ = subprocess.call('python setup.py build_ext --inplace', shell=True)

-->

```
>>> from cos_module import cos_func as cos
>>> cos(1.0)                              
0.5403023058681398

```

---


### Summary

* Input files in the form of triple quotes strings convenient for parameter studies
* The `struct` module has functions for processing binary files
* Binary files from c- and Fortran programs have different structure (Fortran records)
* To speed up Python code write a function in Fortran and compile it with ``f2py`` or use the ``ctypes`` module for C code
-
