# Homepage

For full documentation visit [mkdocs.org](https://www.mkdocs.org).

## Code Annotation Examples

### Codeblocks

Some `code` goes here.

### Plain codeblock

A plain codeblock

```
Some code here
def myfunction()
// some comment
```

#### Code for a specific language 

Some more code with the `py` at the start:

``` py 
import tensorflow as tf
def whatever()
```

#### With a title 

Some more code with the `py` at the start:

``` py title="bluble_sort.py"
# Driver code to test above
arr = [64, 34, 25, 12, 22, 11, 90]
 
bubbleSort(arr)
 
print("Sorted array is:")
for i in range(len(arr)):
    print("% d" % arr[i], end=" ")
```

#### With line numbers

Some more code with the `py` at the start:

``` py linenums="1"
# Driver code to test above
arr = [64, 34, 25, 12, 22, 11, 90]
 
bubbleSort(arr)
 
print("Sorted array is:")
for i in range(len(arr)):
    print("% d" % arr[i], end=" ")
```

#### Highlighting lines

Some more code with the `py` at the start:

``` py linenums="1" hl_lines="2 4"
# Driver code to test above
arr = [64, 34, 25, 12, 22, 11, 90]
 
bubbleSort(arr)
 
print("Sorted array is:")
for i in range(len(arr)):
    print("% d" % arr[i], end=" ")
```