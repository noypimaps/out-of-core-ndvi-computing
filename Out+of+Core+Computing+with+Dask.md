
## Out of Core NDVI Computing with Dask

The following is the implementation specifics of the topic `Out of Core Computing with Dask`. Dask is python library which deals with parallel and distributed computing. It allows processing of data which goes beyond your machine's memory.

For an in-depth tutorial/overview of Dask please refer to the following links:
- http://dask.pydata.org/en/latest/
- https://github.com/mrocklin/dask-pydata-dc-2016
- https://www.youtube.com/watch?v=s4ChP7tc3tA

### VM Specifications
For this notebook, we have setup a virtual machine with the following resources:
- Processors: 2
- RAM: 1GB

We have setup this virtual machine in order for us to simulate a machine with limited capacity and resources.


### Data Specifications
The dataset used was a Landsat 8 satellite image with the following specifications: 
- Bands: 3 and 4
- Filesize: 119.6 mb each
- Arraysize: 7641, 7821

As you can see that our data can be processed by any computer which can be bought from the market. But for the current virtual machine that we have setup, our machine will struggle to process this and may give us an *out of memory* error.

You can download similar Landsat 8 datasets at https://libra.developmentseed.org/


### Implementation Details
The following example will show the typical way of manipulating satellite images in python.

### Rasterio + Numpy
The typical way of manipulating this data is to use gdal/rasterio and numpy. Note that the numpy library utilizes your machines RAM or memory for its processing.


```python
import numpy as np
import rasterio as rio

rio_dataset_red = rio.open('../data/LC81160482016300LGN00_B3.TIF')
rio_dataset_nir = rio.open('../data/LC81160482016300LGN00_B4.TIF')

np_red = rio_dataset_red.read()
np_nir = rio_dataset_nir.read()

ndvi = (np_nir - np_red) / (np_nir + np_red)
ndvi
```


    ---------------------------------------------------------------------------

    MemoryError                               Traceback (most recent call last)

    <ipython-input-1-770ed4093154> in <module>()
          8 np_nir = rio_dataset_nir.read()
          9 
    ---> 10 ndvi = (np_nir - np_red) / (np_nir + np_red)
         11 ndvi


    MemoryError: 


As you have noticed the typical way of reading and manipulating these sizes of data will not work on our virtual machine since we have a very low RAM.


### Dask saves the Day
Dask allows us to manipulate these types of data which will not fit into memory.

Dask does not have an out of the box support for geospatial datasets therefore we will use the following script/implementation which integrates rasterio, dask and numpy. 

The rasterio-dask implementation was created by **Luke Pinner** and can be accessed through the following link: https://gist.github.com/lpinner/bd57b54a5c6903e4a6a2 


```python
import numpy as np
import rasterio as rio
import dask
import dask.array as da

class RioArray(da.Array):
    def __init__(self, filepath, band=1):
        self.dataset = rio.open(filepath)
        blocks = list(self.dataset.block_windows())
        block_shape = self.dataset.block_shapes[band-1]
        chunks = block_shape
        dask = {(filepath,ji[0],ji[1]):
                   (self.dataset.read, band, None, window)
                   for ji, window in blocks
               }

        name = filepath
        dtype = self.dataset.dtypes[band-1]
        shape = self.dataset.shape

        da.Array.__init__(self, dask, name, chunks, dtype, shape)


    def __del__(self):
        try:
            self._dataset.close()
            del self._dataset
        except:pass

```

The *RioArray()* requires the location of your data and the band number you wish to process.

The following example will compute the Normalized Digital Vegetation Index or NDVI which enables researchers and other experts determine if there is a live vegetation or not. In this case, we will use bands 3 (red band) and 4 (near infrared) to compute NDVI and using these bands we assign them to `red` and `nir` respectively.

The formula for computing NDVI:

**NDVI = (nir - red) / (nir + red)**


```python
red = RioArray('../data/LC81160482016300LGN00_B3.TIF', 1)
nir = RioArray('../data/LC81160482016300LGN00_B4.TIF', 1)
```

We can print the red where this will return a Dask object.


```python
red
```




    dask.array<../data/LC81160482016300LGN00_B3.TIF, shape=(7821, 7641), dtype=uint16, chunksize=(1, 7641)>



Applying the formula above:


```python
ndvi = (nir - red) / (nir + red)
ndvi
```

    /home/ubuntu/env/lib/python3.5/site-packages/dask/array/core.py:476: RuntimeWarning: divide by zero encountered in true_divide
      o = func(*args, **kwargs)





    dask.array<truediv, shape=(7821, 7641), dtype=float64, chunksize=(1, 7641)>



Notice that the even though we have applied the formula it will still return the Dask object. Dask will not execute the operation until you have called the *compute()* function. 

**Note:**
The *compute()* function will execute the operation into memory. Executing this method for a resource stricken machine or operations having large results is not advisable.

In the following cell, we will try executing the *compute()* function.


```python
ndvi.compute()
```

    /home/ubuntu/env/lib/python3.5/site-packages/dask/async.py:247: RuntimeWarning: invalid value encountered in true_divide
      return func(*args2)
    /home/ubuntu/env/lib/python3.5/site-packages/dask/async.py:247: RuntimeWarning: divide by zero encountered in true_divide
      return func(*args2)



    ---------------------------------------------------------------------------

    MemoryError                               Traceback (most recent call last)

    <ipython-input-5-ced20570bd01> in <module>()
    ----> 1 ndvi.compute()
    

    /home/ubuntu/env/lib/python3.5/site-packages/dask/base.py in compute(self, **kwargs)
         93             Extra keywords to forward to the scheduler ``get`` function.
         94         """
    ---> 95         (result,) = compute(self, traverse=False, **kwargs)
         96         return result
         97 


    /home/ubuntu/env/lib/python3.5/site-packages/dask/base.py in compute(*args, **kwargs)
        205     return tuple(a if not isinstance(a, Base)
        206                  else a._finalize(next(results_iter))
    --> 207                  for a in args)
        208 
        209 


    /home/ubuntu/env/lib/python3.5/site-packages/dask/base.py in <genexpr>(.0)
        205     return tuple(a if not isinstance(a, Base)
        206                  else a._finalize(next(results_iter))
    --> 207                  for a in args)
        208 
        209 


    /home/ubuntu/env/lib/python3.5/site-packages/dask/array/core.py in finalize(results)
        914     while isinstance(results2, (tuple, list)):
        915         if len(results2) > 1:
    --> 916             return concatenate3(results)
        917         else:
        918             results2 = results2[0]


    /home/ubuntu/env/lib/python3.5/site-packages/dask/array/core.py in concatenate3(arrays)
       3344             return type(x)
       3345 
    -> 3346     result = np.empty(shape=shape, dtype=dtype(deepfirst(arrays)))
       3347 
       3348     for (idx, arr) in zip(slices_from_chunks(chunks), core.flatten(arrays)):


    MemoryError: 


Notice that dask will produce a *MemoryError* since the result will utilize more than 1GB of memory.

In these cases, where the size of the result is more than the size of the allocated memory we will use the disk space of our machine. The *compute()* function is similar with how use a typical numpy array.

Dask provides us a mechanism where we can store our result onto disk where we have more than enough.

In the following cell, we will save the result of our operations to disk using *to_hdf5()*. This function stores the result of our NDVI computation into hdf5.


```python
# store the resultant/intermediate arrays onto disk
da.to_hdf5('ndvi_result.hdf5', '/ndvi', ndvi)
```

    /home/ubuntu/env/lib/python3.5/site-packages/dask/async.py:247: RuntimeWarning: invalid value encountered in true_divide
      return func(*args2)
    /home/ubuntu/env/lib/python3.5/site-packages/dask/async.py:247: RuntimeWarning: divide by zero encountered in true_divide
      return func(*args2)


Notice that we didn't have any problem with memory. I should be noted though that storing our computation to disk may create additional latency. 

Let us inspect the created hdf5 file. Notice the file size generated.


```python
import os
statinfo = os.stat('ndvi_result.hdf5')
statinfo.st_size
```




    478494200



The result is around 478mb.

It can be said that the filesize can be stored into 1GB RAM but it should be noted that the operating system and other processes also takes up RAM so loading this file into memory may introduce *MemoryError* for a resource stricken machine.
