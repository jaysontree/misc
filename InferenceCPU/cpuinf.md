# Pracitice: Inference Server with CPU

### intro

CPU inferencing Service:
- low cost
- high concurrency, easy to scale up


##### compuation capacity
CPU Peak Performance: `num of cores` *x* `clock` *x* `num of computes(instruction set)`

<font size='2' color='purple'>--- more detail on CPU Benchmark: [cpufp](https://github.com/pigirons/cpufp) </font>

Memory Bandwidth: `data transportaion per clock cycle` *x* `clock` *x* `memory interface width`

##### model complexity
time complexity `F`: num of M-Adds (* precision)

space complexity `B`: num of Params (* precision)

Operation intensity: `I=F/B` 

##### ideal computation time cost:
`t = total_F / min(I * Bandwidth, CPU Peak Performance)`

##### CPU inference Acceleration
- fit the best instruction set
- intraOP parallelism 

<font size=2 color='purple'> --- more details: see MKLdnn, MLAS </font>

### Experiment
##### Single worker Experiment
Exp1: Accelerate with higher intraOP parallelism
Test Case: Text Recognition Service
several cores serve one request
- CPU: Intel Xeon Gold 6226R (32 cores)
- variable: OMP_NUM_THREADS

| intraOP Parallelism | MKL verbose | CPU utilization | RPS |
| ----- | ----- | ----- | ----- |
| 1 | NThr: 1 | 104% | 3.658 |
| 2 | NThr: 2 | 206% | 5.042 |
| 4 | NThr: 4 | 405% | 7.099 |
| 8 | NThr: 8 | 810% | 9.222 |
| 16 | NThr: 16 | 1590% | 9.316 |
| 32* | NThr: 32 | 2683~3940% | 1.393* | 

observed： 
- nonlinear incremental with higher parallelism
- adding intraOP parallelism after certain point wastes resource 
<font size=2> *other load on the machine caused block </font> 
<font size=2> for containered deployment, if CPU resource is limited by time slicing, the performance of large parallelism may dramatically slow down due to thread block</font>

Exp2: CPU performance cap
Test Case: simple convolution net
- CPU: Intel i7-7700k (4 cores)
- variable: OMP_NUM_THREADS, KMP_AFFINITY

| intraOP Parallelism | MKL verbose | CPU utilization | RPS |
| ----- | ----- | ----- | ----- |
| 8 | NThr: 8 | 800% | 8.25 |
| 4 | NThr: 4 | 400% | 8.29 |
| 4 with thread-core bounding | NThr: 4 | 400% | __8.46__ |

observed：
- parallelism acceleration is bounded by num of physical cores of CPU
- in such computing tasks, applying more threads does not improve efficiency and  thread switching has cost

:speech_balloon: <font color='blue'>In single worker case, increaing intraOP parallelism can reduce infernce time for many nets. the best practice is to have num of `intraOP parallelism = num of cores` and bound thread and core.</font>

##### Multi-woker concurrency Experiment
Exp3: 
Test Case: Text Recognition Service
Multi cores serves multi requests
- CPU: Intel i7-7700k (4cores)
- variable: num of gunicorn workers/num of replica, OMP_NUM_THREADS

| workers/ replicas | intraOP paral | total | low load CPU utilization | high load CPU utilization | low load response time(ms) | high load response time(ms) | high load RPS |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| 4 | 1 | 4 | 99% | 395% | 211 | 266 | __15.90__ |
| 4 | 4 | 16 | 399% | 787% | 79 | 389 | 10.20 |
| 1 | 4 | 4 | 406% | 402% | 80 | 330 | 12.00 |

![response time vs load](response_load.jpg "Y:response time, X: load")

observed:
- In low load situation, the result is similar to single worker experiment. having the most intraOP parallelism lead to lowest respose time.
- In high load situation, workers on the same machine may compete for resource. parallelism exceeding num of CPU cores may may slow down the service. outliers with extrodinary high response time are observed, which may be casued by thread block, making the service unstable.
- using concurrent workers with no intraOP parrallelism is most stable if load increases, and most efficient under high load. 



:speech_balloon: <font color='blue'>In concurrency case, if response time is important, it is better to have intraOP parallelism simiar to single worker case. While repicas with no intraOP parallelism have the best RPS, almost 30% increase in the test case. Other factors such as memory usage shall also be concerned to decide whether use replicas or intraOP parallelism. A probably good pratice is to have `num of replicas * intraOP parallelism = num of cores`</font>
