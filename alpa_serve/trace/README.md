# Trace Replay


## Dataset
This folder provides methods to generate a TraceReplay from a public trace. Supported public trace:
- Microsoft azure_v1 trace. [[Intrduction]](https://github.com/Azure/AzurePublicDataset/blob/master/AzureFunctionsDataset2019.md) [[Download]](https://drive.google.com/file/d/1Kup6JUH523CZZ7OxlkO942nAd5opuro0/view?usp=sharing)
- Microsoft azure_v2 trace. [[Introduction]](https://github.com/Azure/AzurePublicDataset/blob/master/AzureFunctionsInvocationTrace2021.md) [[Download]](https://drive.google.com/file/d/1IOVoUoodBj4aKeyggxMnEVChEPutN4t7/view?usp=sharing)

以下过程都需要预处理之后的pkl文件，但是上面两个下载数据的超链接指向的谷歌云盘并没有开放访问权限。重新构建pkl文件的方法在trace.py里的preprocess_azure_v1_trace函数(~7 min, 5.5 GB)、preprocess_azure_v2_trace函数(~1 min, 16 MB)。

replay把指定时间段内的请求映射到给定的模型上，分为round_robin和stripe，就是一个一个分和一块一块分。

## How to use
First construct a trace object, which will read one of the two traces:
```python
trace_name = "azure_v2"
trace_dir = "~/azure_v2.pkl"
trace = Trace(trace_name, trace_dir)
```
Provide a model that you want the trace to be replayed for:
```python
n_model = 5
models = [f"gpt{i}" for i in range(n_model)]
```


Replay the vanilla `azure_v2` trace in day 1. `azure_v1` cannot be replayed in vanilla mode. 
```python

replays = trace.replay_vanilla(models,
                               model_mapping_strategy="stripe",
                               start_time="0.0.0",
                               end_time="1.0.0")
```

Replay `azure_v2` trace in day 1 - 5. Estimate a Gamma arrival distribution using the data from each 3600-second window 
and sample the arrivals from Gamma distributions.
```python
replays = trace.replay(models,
                       model_mapping_strategy="stripe",
                       start_time="0.0.0",
                       end_time="5.0.0",
                       arrival_distribution="gamma",
                       interval_seconds=3600)
```

Replay the vanilla `azure_v2` trace in day 1 - 14. However, scale the trace as if they happened in 7 days.
```python
replays = trace.replay(models,
                       model_mapping_strategy="stripe",
                       start_time="0.0.0",
                       end_time="13.23.60",
                       arrival_distribution="vanilla",
                       time_scale_factor=2.0)
```

Replay the `azure_v2` trace in day 1 using a Gamma estimator. But scale the Gamma distributions' rate and CV by 8x:
```python
replays = trace.replay(models,
                       model_mapping_strategy="stripe",
                       start_time="0.0.0",
                       end_time="1.0.0",
                       arrival_distribution="gamma",
                       rate_scale_factor=8.0,
                       cv_scale_factor=8.0)
```

You can visualize the replayed trace by:
```python
replays[model_name].report_stats()
replays[model_name].visualize()
```

You can convert a TraceReplay to be a workload:
```python
replays[model_name].to_workload(slo=1.0)
```
