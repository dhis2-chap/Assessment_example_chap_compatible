
NOTE! This tutorial has been updated. Go the the main branch to see the old version.

# CHAP-compatible example of model assessment 
This tutorial provides a guide to and examples of how to do develop a new (custom) evaluation metric that can be used for model evaluation in CHAP.


## Some background on evaluation metrics

When evaluating a chap-compatible model with chap, the model will give some prediction `samples` for every `time_period` (e.g. a week in some year) for every `location` (e.g. a district in a country). An important detail is that this is done for different train-test **split points** in the dataset. For each such split point, the model will predict a certain number of periods (e.g. weeks a head).

This means that every predicted disease case can be tied to four variables:

- `location`
- `time_period`
- `horizon_distance`  (how far from a split point was this prediction made)
- `sample` (just an index for the sample, if the model gives 10 predictions for this location/time_period/horizon_distance, then this will go from 0 to 9)

When dealing with metrics in chap, we represent all this information using a "flat" pandas dataframe. Below is an example of the predictions given by a model for two different locations two weeks ahead:

```
  location time_period  horizon_distance  sample  forecast
0     loc1    2023-W01                 1       0        10
1     loc1    2023-W02                 2       0        12
2     loc2    2023-W01                 1       0        21
3     loc2    2023-W02                 2       0        23
```

From the above data, we can see that the model just gave one sample for each location/time_period/horizon_distance combination. Also, there was only one split point (and two horizon distances, meaning the model predicted two weeks ahead). Note that all this could vary based on the evaluation setup and the model.

The "true" observations can be represented in a similar way, except that we don't need to represent the sample index or horizon distance for true observations:

```
  location time_period  disease_cases
0     loc1    2023-W01           11.0
1     loc1    2023-W02           13.0
2     loc2    2023-W01           19.0
3     loc2    2023-W02           21.0
```

### Metrics in chap

Metrics in chap are in principle functions that take observed disease cases and predicted cases in the format shown above and returns a dataframe with the metric.

The output format is a dataframe with columns corresponding to what "level of detail" the metric has been computed for. For instance, if a metric is computed for each location, the output columns will be "location" and "metric", e.g:

```
    location  metric
0      loc1    1.0
1      loc2    2.0
```

However, a metric is free to aggregate over locations, time_periods or horizon_distance as it sees fit. For instance, a metric that only aggregates over time_periods might give this output:

```
    location  horizon_distance  metric
0      loc1                 1    0.5
1      loc2                 1    0.6
2      loc1                 2    0.7
3      loc2                 2    0.8
```


## Isolated example: Starting by implementing a simple metric outside of chap

Since a metric only depends on these simple pandas dataframes, it is easy to implement new metrics as a function outside of chap. 
This is useful for testing and debugging. Later in this guide, we show how to move a metric inside chap so that it can be used in the platform (e.g. to generate plots in the modeling app). 
This only requires implementing the metric function in a class that follows a interface.

This example only requires that you have pandas installed.

```python
import pandas as pd
    
forecasts = pd.DataFrame(
    {
        "location": ["loc1", "loc1", "loc2", "loc2"],
        "time_period": ["2023-W01", "2023-W02", "2023-W01", "2023-W02"],
        "horizon_distance": [1, 2, 1, 2],
        "sample": [1, 1, 1, 1],
        "forecast": [10, 12, 21, 23],
    }
)


observations = pd.DataFrame(
    {
        "location": ["loc1", "loc1", "loc2", "loc2"],
        "time_period": ["2023-W01", "2023-W02", "2023-W01", "2023-W02"],
        "disease_cases": [11.0, 13.0, 19.0, 21.0],
    }
)


def my_metric(forecasts: pd.DataFrame, observations: pd.DataFrame) -> pd.DataFrame:
    # sum of absolute error per location and time_period
    merged = forecasts.merge(observations, on=["location", "time_period"], how="left")
    merged["metric"] = (merged["forecast"] - merged["disease_cases"]).abs()
    return merged[["location", "time_period", "metric"]]

print(my_metric(forecasts, observations))
```

This code can also be found in isolated_asses.py. Feel free to play around with it and try to implement other metrics. The output from running `isolated_asses.py` should look like this:

```
  location time_period  metric
0     loc1    2023-W01     1.0
1     loc1    2023-W02     1.0
2     loc2    2023-W01     2.0
3     loc2    2023-W02     2.0
```


## Implementing a custom metric that is compatible with chap
Currently, metrics are implemented in chap-core in the assessment/metrics/ directory. A valid metric subclasses the MetricBase class and implements a compute method.

The compute method is similar to a function implemented in the isolated_asses.py example above, but in chap-core we can take use of some typing to ease development.

In the example_metric.py file, we have implemented the metric above in an ExampleMetric class. The code can also be found working inside chap-core in [example_metric.py](https://github.com/dhis2-chap/chap-core/blob/master/chap_core/assessment/metrics/example_metric.py) assessment/metrics/.

Note the metric_spec variable. This is important in order to tell chap what the metric outputs:

```python
# ...
spec = MetricSpec(
        output_dimensions=(DataDimension.location, DataDimension.time_period),
        metric_name="Example Absolute Error",
        metric_id="example_metric",
        description="Sum of absolute error per location and time_period",
    )
# ...
```

If you run `get_metric` on this class, chap will check that the output actually matches the spec. If for instance, you forget to specify location, you will get this error message from chap:

```
ValueError: ExampleMetric produced wrong columns.
Expected: [<DataDimension.time_period: 'time_period'>, 'metric']
Missing: []
Extra: ['location']
```



## Adding the metric to the chap-core codebase
After succesfully implementing a chap-compatible metric, all that is needed in order to use it in chap is to add it to the available_metrics registry in chap-core/assessment/metrics/__init__.py. Typically, your metric would be in a separate file in the assessment/metrics/ directory, and you would import it in the __init__.py file and add it to the available_metrics dictionary.

If the metric is compatible with plotting (valid output dimensions), it will automatically be available as a plotting option in the modeling app in chap.


