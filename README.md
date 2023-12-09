# DrPy: Pinpointing Inefficient Memory Usage in Multi-Layer
**Dy**namical profiling tools for **Py**thon applications.

This repository contains explanations for the artifact evaluations for our paper *DrPy: Pinpointing Inefficient Memory Usage in Multi-Layer*.  The **DrPy** tool generates value flow graphs to help programmers pinpoint where the redundant copy of memory objects happens. We utilize the tool **DrPy** to help find potential optimization opportunities, especially targeting the redundant copy in Python applications.

## Getting Started Guide

### Run with docker

1. Pull the docker image
```shell
docker pull captaincui/drpy
```

2. Run the docker as a container
```shell
docker run -t -d --name drpy-dev captaincui/drpy
```

3. Run zsh shell with the preset user `ubuntu`
```shell
docker exec -it --user ubuntu drpy-dev zsh
```


### Tool Functionality

The following examples help demonstrate how to utilize the tool to eliminate redundancies.


#### I. numpy.array()

According to the documentation of the numpy API, [numpy.array(object, dtype=None, *, copy=True, order='K', subok=False, ndmin=0, like=None)](https://numpy.org/doc/stable/reference/generated/numpy.array.html), the parameter `copy` defaults to `True` and therefore creates a copy of the argument `object`.

The code snippets in the `example` folder demonstrates the fact:

```shell
# Activate the conda environment with Numpy installed
conda activate simpletest

cd ~/workspace/example
```

The Python program `np_array.py` generates a Numpy array `random_array` and then passes it through several functions. Some functions only read the array. The function `func_make_a_copy` uses `np.array()` API to make a copy of the array `random_array`.

```Python
...
def func_make_a_copy(arr):
    print(f"Before copy, array mem address:   {id(arr)}")
    
    # np.array() defacuts to create a copy of the provied obj arr
    array_copy = np.array(arr)                              
    # array_copy = arr.copy()     # alternative way to copy

    print(f"After copy, copied array address: {id(array_copy)}")
    func_modify_the_copy(array_copy)
...
```

Run the script with the following command,
```shell
python np_array.py
```

The output will be similar to this:
```Python
id(random_array):                 139870089688624
id(only_read_array):              139870089688624
Before copy, array mem address:   139870089688624
After copy, copied array address: 139870348654320
Mem address of the copy:          139870348654320
```
The Python built-in function `id()` returns the memory address of Python objects in the Python interpreter.

The first three lines indicate when `random_array` is passed by reference, the memory addresses of derived Python objects stay the same. At the 4th line, the program prints out the address of the copied object. The changed address indicates the copied object does not share the same memory as the `random_array` object. The last line authenticates that objects passed by reference are as similar as alias, their memory address are the same.

However, the copy in the function `func_make_a_copy` is unnecessary, or, **redundant**, because there is no further modification on the copied object. The subsequent function can use reference as well.

##### Step 1: Detect the Copy
When we apply **DyPy** tool on this script, it can flag the redundant copy:

```shell
$drrun -t drcctlib_python -- python -m valueflow np_array.py 
```
> **Note**: Please prefix `drrun` with the `$`. The `drrun` is a pre-defined environment variable. It is set at the last line in the `.zshrc` file locates in the `/home/ubuntu` directory.
```bash
export drrun=/home/ubuntu/workspace/drcctprof_clients/DrCCTProf/build/bin64/drrun
```
> The above **DrPy** tool command is the same as follows:
```shell
/home/ubuntu/workspace/drcctprof_clients/DrCCTProf/build/bin64/drrun -t drcctlib_python -- python -m valueflow np_array.py 
```

The **DrPy** tool command will generate 2 image files with the same name as the application `np_array.py` suffixed with `.png` and `.svg`, respectively. The `.png` file is easy to open for a quick skim, and the `.svg` file is suitable for scaling up for more detailed information.

The figure below is the value flow graph for the `np_array.py` example script:

![np_array value flow graph](images/np_array.py.png)

The red arrow indicates there exist redundancies when values flow through the function `func_pass_it_through` to the function `func_make_a_copy`. If you substitute the `np.array(arr)` at Line 16 with `arr.copy()` and run the command again, you will get the same value flow graph.

##### Step 2: No redundant, no flag
Let's copy the Python script `np_array.py` to `no_copy.py`
```shell
cp np_array.py no_copy.py
```
Open file `no_copy.py` and comment code at line 15 `array_copy = np.array(arr)`. Add another line of code beneath line 15:
```Python
14    # np.array() defacuts to create a copy of the provied obj arr
15    # array_copy = np.array(arr)                              
16    array_copy = arr
17    # array_copy = arr.copy()         # arr.copy() has the same effect
```

Run **DrPy** tool on **no_copy.py**
```shell
$drrun -t drcctlib_python -- python -m valueflow no_copy.py 
```

The value flow graph updates as the following picture:

![no_copy value flow graph](images/no_copy.py.png)

There is no red arrow between the aforementioned two functions, as there is no redundant copy.

##### Step 3: No false positive
Let's copy the original Python script `np_array.py` to `edit_copy.py`

```shell
cp np_array.py edit_copy.py
```
Open the file `edit_copy.py` and focus on the function `func_modify_the_copy` at line 21. Add a new line beneath line 21:

```Python
21 def func_modify_the_copy(arr):
22    arr[0] = 3.14
23    func_read_the_copy(arr)
```
Run **DrPy** tool on `edit_copy.py`:
```shell
$drrun -t drcctlib_python -- python -m valueflow edit_copy.py
```

The corresponding value flow graph is as the picture beblow:

![edit_copy value flow graph](images/edit_copy.py.png)

There is no red warning on the graph. It is reasonable because the program in `edit_copy.py` modifies the copied object after the copy operation. The copy is not regarded as redundant because the programmer might want to keep the original data for future use.

<!-- 4. (Optional) Build the `DrPy` tool
```shell
cd ~/workspace/drcctprof_clients
./build_clean.sh
./build.sh
``` -->

