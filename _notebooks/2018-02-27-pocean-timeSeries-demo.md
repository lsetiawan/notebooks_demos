---
title: "Creating a CF-1.6 timeSeries using pocean"
layout: notebook

---


OOS recommends to data providers that their netCDF files follow the CF-1.6 standard. In this notebook we will create a [CF-1.6 compliant](http://cfconventions.org/latest.html) file that follows file that follows the [Discrete Sampling Geometries](http://cfconventions.org/Data/cf-conventions/cf-conventions-1.7/build/ch09.html) (DSG) of a `timeSeries` from a pandas DataFrame.

The `pocean` module can handle all the DSGs described in the CF-1.6 document: `point`, `timeSeries`, `trajectory`, `profile`, `timeSeriesProfile`, and `trajectoryProfile`. These DSGs array may be represented in the netCDF file as:

- **orthogonal multidimensional**: when the coordinates along the element axis of the features are identical;
- **incomplete multidimensional**: when the features within a collection do not all have the same number but space is not an issue and using longest feature to all features is convenient;
- **contiguous ragged**: can be used if the size of each feature is known;
- **indexed ragged**: stores the features interleaved along the sample dimension in the data variable.

Here we will use the orthogonal multidimensional array to represent time-series data from am hypothetical current meter. We'll use fake data for this example for convenience.

Our fake data represents a current meter located at 10 meters depth collected last week.

<div class="prompt input_prompt">
In&nbsp;[1]:
</div>

```python
from datetime import datetime, timedelta

import numpy as np
import pandas as pd


x = np.arange(100, 110, 0.1)
start = datetime.now() - timedelta(days=7)

df = pd.DataFrame({
    'time': [start + timedelta(days=n) for n in range(len(x))],
    'longitude': -48.6256,
    'latitude': -27.5717,
    'depth': 10,
    'u': np.sin(x),
    'v': np.cos(x),
    'station': 'fake buoy',
})


df.tail()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>depth</th>
      <th>latitude</th>
      <th>longitude</th>
      <th>station</th>
      <th>time</th>
      <th>u</th>
      <th>v</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>95</th>
      <td>10</td>
      <td>-27.5717</td>
      <td>-48.6256</td>
      <td>fake buoy</td>
      <td>2018-05-25 19:10:29.519535</td>
      <td>0.440129</td>
      <td>-0.897934</td>
    </tr>
    <tr>
      <th>96</th>
      <td>10</td>
      <td>-27.5717</td>
      <td>-48.6256</td>
      <td>fake buoy</td>
      <td>2018-05-26 19:10:29.519535</td>
      <td>0.348287</td>
      <td>-0.937388</td>
    </tr>
    <tr>
      <th>97</th>
      <td>10</td>
      <td>-27.5717</td>
      <td>-48.6256</td>
      <td>fake buoy</td>
      <td>2018-05-27 19:10:29.519535</td>
      <td>0.252964</td>
      <td>-0.967476</td>
    </tr>
    <tr>
      <th>98</th>
      <td>10</td>
      <td>-27.5717</td>
      <td>-48.6256</td>
      <td>fake buoy</td>
      <td>2018-05-28 19:10:29.519535</td>
      <td>0.155114</td>
      <td>-0.987897</td>
    </tr>
    <tr>
      <th>99</th>
      <td>10</td>
      <td>-27.5717</td>
      <td>-48.6256</td>
      <td>fake buoy</td>
      <td>2018-05-29 19:10:29.519535</td>
      <td>0.055714</td>
      <td>-0.998447</td>
    </tr>
  </tbody>
</table>
</div>



Let's take a look at our fake data.

<div class="prompt input_prompt">
In&nbsp;[2]:
</div>

```python
%matplotlib inline


import matplotlib.pyplot as plt
from oceans.plotting import stick_plot

q = stick_plot(
    [t.to_pydatetime() for t in df['time']],
    df['u'],
    df['v']
)

ref = 1
qk = plt.quiverkey(q, 0.1, 0.85, ref,
                  "%s m s$^{-1}$" % ref,
                  labelpos='N', coordinates='axes')

_ = plt.xticks(rotation=70)
```


![png](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAWQAAAEsCAYAAAD0EMkhAAAABHNCSVQICAgIfAhkiAAAAAlwSFlzAAALEgAACxIB0t1+/AAAADl0RVh0U29mdHdhcmUAbWF0cGxvdGxpYiB2ZXJzaW9uIDIuMS4yLCBodHRwOi8vbWF0cGxvdGxpYi5vcmcvNQv5yAAAIABJREFUeJzt3XdcVeUfB/DP4whHuXAkmpKpmWbLUZZtAxwI4lbclutn5p5oWmI50zS3kqGWZoapCO4caG4RcySgOBlOUFn38/vjXJ44aLbEe7Dv+/XiJee5V+6Xw7mf+5znPOccRRJCCCEcL5ejCxBCCGGQQBZCCIuQQBZCCIuQQBZCCIuQQBZCCIuQQBZCCIuQQBZCCIuQQBZCCIuQQBZCCIuQQBZCCIvI83eeXLx4cbq6umZTKUII8XDat29fPMkSf/a8vxXIrq6u2Lt37z+vSggh/oOUUqf/yvNkyEIIISxCAlkIISxCAlkIISxCAlkIISxCAlkIISxCAlkIISxCAlkIISxCAlkIISxCAlkIISxCAlkIISxCAlkIISziPxPIkZGR6NKlC5o1a+boUoQQ4q4eWCB37twZJUuWxLPPPvugXtKkQoUKmD9/vkNeWwgh/ooHFsgdO3bEunXrsv11wsPD0ahRI9NXbGxstr+uEEL8W3/r8pv/xhtvvIHo6Og/fDw6OhoeHh6oW7cudu3aheeffx6dOnXCqFGjEBsbi8WLF6N27dqm/5OUlIQWLVrg7NmzSE9Ph5+fH1q2bInVq1dn828jhBD3n6XGkH/77Tf06dMHhw8fxrFjx7BkyRJs374dEydOhL+//x3PX7duHVxcXHDo0CEcOXIEHh4ef/izExIS0L17dxw4cADjxo3Lzl9DCCH+kQfWQ/4rnnzySVSvXh0AUK1aNbz77rtQSqF69ep37V1Xr14dAwYMwODBg9GoUSO8/vrrf/iznZ2dMWvWrOwqXQgh/jVL9ZCdnJz097ly5dLLuXLlQlpa2h3Pr1y5Mvbt24fq1atj6NChGDNmzAOrVQgh7jdL9ZD/rvPnz6NYsWLw9fXFo48+ioCAAEeXJIQQ/9gDC+TWrVtjy5YtiI+PR9myZTF69Gh06dLlX/3M8PBwDBw4ELly5ULevHkxc+bM+1StEEI8eIrkX35yzZo1KTc5FUKIv0cptY9kzT97nqXGkIUQ4r9MAlkIISxCAlkIISxCAlkIISxCAlkIISxCAlkIISxCAlkIISxCAlkIISxCAlkIISzCUteyUErdl5/zd84+FEIIq7BUIEuQCiH+y2TIQgghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLEICWQghLMJSgZySkoL09HRTW3R09B1tMTExD7IsIR6oxMREpKWlmdqioqLueB9cuHDhQZYlHoAHEshJSUk4f/68qW3Tpk3Yvn27qW38+PE4dOiQqc3f3x/Xrl0ztfXt2/eO1/D19TUtk0RAQMAdbTab7e+WL8R9kZCQgN9++83UtmbNGmzZssXUNnnyZBw5csTU9umnnyIpKcnU1rt37zte4/333zctk8TKlSv/RdXiQbrvgZyeno5FixaZ2q5evYqRI0ea2kqUKIGQkBBTW4UKFRAVFWVqy5cvH27fvn3P17x169YdQRseHo6TJ0+a2rZt24apU6ea2s6fP4/r16/f8+cL8XelpqZi0qRJpraUlBR89tlnpjYXFxds3brV1FauXLk79gJz5859R685qxs3biA5OdnUdvDgQezfv9/UtnPnTixYsOCO/yudFcf714E8efJkXL16VS/nzp0bwcHBuHTpkm4rU6YM4uPjTRtLtWrV7ugFVKhQAZGRkaa2vxLIx44dwzPPPGNqCwkJgbu7u6ktICAALVq0MLUNGjTIVD8A7Nq1CyTv+ZpCZDZw4EDExcXp5bx582Lfvn2m90Hp0qURFxeH1NRU3Va9enWEh4ebftYTTzyBM2fOmNry5Mnzp4EcHh6O5557ztS2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
)


`pocean.dsg` is relatively simple to use. The user must provide a DataFrame, like the one above, and a dictionary of attributes that maps to the data and adhere to the DSG conventions desired. 

Because we want the file to work seamlessly with ERDDAP we also added some ERDDAP specific attributes like `cdm_timeseries_variables`, and `subsetVariables`.

<div class="prompt input_prompt">
In&nbsp;[3]:
</div>

```python
attributes = {
    'global': {
        'title': 'Fake mooring',
        'summary': 'Vector current meter ADCP @ 10 m',
        'institution': 'Restaurant at the end of the universe',
        'cdm_timeseries_variables': 'station',
        'subsetVariables': 'depth',
    },
    'longitude': {
        'units': 'degrees_east',
        'standard_name': 'longitude',
    },
    'latitude': {
        'units': 'degrees_north',
        'standard_name': 'latitude',
    },
    'z': {
        'units': 'm',
        'standard_name': 'depth',
        'positive': 'down',
    },
    'u': {
        'units': 'm/s',
        'standard_name': 'eastward_sea_water_velocity',
    },
    'v': {
        'units': 'm/s',
        'standard_name': 'northward_sea_water_velocity',
    },
    'station': {
        'cf_role': 'timeseries_id'
    },
}
```

We also need to map the our data axes to [`pocean`'s defaults](https://github.com/pyoceans/pocean-core/blob/master/pocean/utils.py#L50-L59). This step is not needed if the data axes are already named like the default ones.

<div class="prompt input_prompt">
In&nbsp;[4]:
</div>

```python
axes = {'t': 'time', 'x': 'longitude', 'y': 'latitude', 'z': 'depth'}
```

<div class="prompt input_prompt">
In&nbsp;[5]:
</div>

```python
from pocean.dsg.timeseries.om import OrthogonalMultidimensionalTimeseries
from pocean.utils import downcast_dataframe


df = downcast_dataframe(df)  # safely cast depth np.int64 to np.int32
dsg = OrthogonalMultidimensionalTimeseries.from_dataframe(
    df,
    output='fake_buoy.nc',
    attributes=attributes,
    axes=axes,
)
```

The `OrthogonalMultidimensionalTimeseries` saves the DataFrame into a CF-1.6 TimeSeries DSG.

<div class="prompt input_prompt">
In&nbsp;[6]:
</div>

```python
!ncdump -h fake_buoy.nc
```
<div class="output_area"><div class="prompt"></div>
<pre>
    netcdf fake_buoy {
    dimensions:
    	time = 100 ;
    	station = 1 ;
    variables:
    	int crs ;
    	string station(station) ;
    		station:cf_role = "timeseries_id" ;
    		station:long_name = "station identifier" ;
    	double time(time) ;
    		time:units = "seconds since 1990-01-01 00:00:00Z" ;
    		time:standard_name = "time" ;
    		time:axis = "T" ;
    	double latitude(station) ;
    		latitude:axis = "Y" ;
    		latitude:units = "degrees_north" ;
    		latitude:standard_name = "latitude" ;
    	double longitude(station) ;
    		longitude:axis = "X" ;
    		longitude:units = "degrees_east" ;
    		longitude:standard_name = "longitude" ;
    	int depth(station) ;
    		depth:_FillValue = -9999 ;
    		depth:axis = "Z" ;
    	double u(station, time) ;
    		u:_FillValue = -9999.9 ;
    		u:units = "m/s" ;
    		u:standard_name = "eastward_sea_water_velocity" ;
    		u:coordinates = "time depth longitude latitude" ;
    	double v(station, time) ;
    		v:_FillValue = -9999.9 ;
    		v:units = "m/s" ;
    		v:standard_name = "northward_sea_water_velocity" ;
    		v:coordinates = "time depth longitude latitude" ;
    
    // global attributes:
    		:Conventions = "CF-1.6" ;
    		:date_created = "2018-02-26T22:10:00Z" ;
    		:featureType = "timeseries" ;
    		:cdm_data_type = "Timeseries" ;
    		:title = "Fake mooring" ;
    		:summary = "Vector current meter ADCP @ 10 m" ;
    		:institution = "Restaurant at the end of the universe" ;
    		:cdm_timeseries_variables = "station" ;
    		:subsetVariables = "depth" ;
    }

</pre>
</div>
 It also outputs the dsg object for inspection. Let us check a few things to see if our objects was created as expected. (Note that some of the metadata was "free" due t the built-in defaults in `pocean`.

<div class="prompt input_prompt">
In&nbsp;[7]:
</div>

```python
dsg.getncattr('featureType')
```




    'timeseries'



<div class="prompt input_prompt">
In&nbsp;[8]:
</div>

```python
type(dsg)
```




    pocean.dsg.timeseries.om.OrthogonalMultidimensionalTimeseries



In addition to standard `netCDF4-python` object `.variables` method `pocean`'s DSGs provides an "categorized" version of the variables in the `data_vars`, `ancillary_vars`, and the DSG axes methods.

<div class="prompt input_prompt">
In&nbsp;[9]:
</div>

```python
[(v.standard_name) for v in dsg.data_vars()]
```




    ['eastward_sea_water_velocity', 'northward_sea_water_velocity']



<div class="prompt input_prompt">
In&nbsp;[10]:
</div>

```python
dsg.axes('T')
```




    [<class 'netCDF4._netCDF4.Variable'>
     float64 time(time)
         units: seconds since 1990-01-01 00:00:00Z
         standard_name: time
         axis: T
     unlimited dimensions: 
     current shape = (100,)
     filling on, default _FillValue of 9.969209968386869e+36 used]



<div class="prompt input_prompt">
In&nbsp;[11]:
</div>

```python
dsg.axes('Z')
```




    [<class 'netCDF4._netCDF4.Variable'>
     int32 depth(station)
         _FillValue: -9999
         axis: Z
     unlimited dimensions: 
     current shape = (1,)
     filling on]



<div class="prompt input_prompt">
In&nbsp;[12]:
</div>

```python
dsg.vatts('station')
```




    {'cf_role': 'timeseries_id', 'long_name': 'station identifier'}



<div class="prompt input_prompt">
In&nbsp;[13]:
</div>

```python
dsg['station'][:]
```




    array(['fake buoy'], dtype=object)



<div class="prompt input_prompt">
In&nbsp;[14]:
</div>

```python
dsg.vatts('u')
```




    {'_FillValue': -9999.9,
     'coordinates': 'time depth longitude latitude',
     'standard_name': 'eastward_sea_water_velocity',
     'units': 'm/s'}



We can easily round-trip back to the pandas DataFrame object.

<div class="prompt input_prompt">
In&nbsp;[15]:
</div>

```python
dsg.to_dataframe().head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>t</th>
      <th>x</th>
      <th>y</th>
      <th>z</th>
      <th>station</th>
      <th>u</th>
      <th>v</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2018-02-19 19:10:29.519535</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>fake buoy</td>
      <td>-0.506366</td>
      <td>0.862319</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2018-02-20 19:10:29.519535</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>fake buoy</td>
      <td>-0.417748</td>
      <td>0.908563</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2018-02-21 19:10:29.519535</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>fake buoy</td>
      <td>-0.324956</td>
      <td>0.945729</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2018-02-22 19:10:29.519535</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>fake buoy</td>
      <td>-0.228917</td>
      <td>0.973446</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2018-02-23 19:10:29.519535</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>fake buoy</td>
      <td>-0.130591</td>
      <td>0.991436</td>
    </tr>
  </tbody>
</table>
</div>



For more information on `pocean` please check the [API docs](https://pyoceans.github.io/pocean-core/docs/api/pocean.html).
<br>
Right click and choose Save link as... to
[download](https://raw.githubusercontent.com/ioos/notebooks_demos/master/notebooks/2018-02-27-pocean-timeSeries-demo.ipynb)
this notebook, or click [here](https://beta.mybinder.org/v2/gh/ioos/notebooks_demos/master?filepath=notebooks/2018-02-27-pocean-timeSeries-demo.ipynb) to run a live instance of this notebook.