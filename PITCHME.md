@title[Introduction]

# ML and Big Data scheduling in Docker cluster management systems

MaSe - Peter Kurfer

<small>Matr.-Nr. 817418</small>

---

## Agenda

- Motivation
- Classic job scheduling systems - Slurm |
- Comparison Slurm vs. Kubernetes |

---

## Motivation

- Docker containers are omnipresent
- Are Docker containers also useful for ML and Big Data processing? |
- Is it possible to use current Docker cluster management systems for job scheduling in multi-user environments? |
- Is it possible to use one or multiple GPUs to increase the performance? |
- Is it possible to run multiple jobs in parallel without them interfering with each other? |

---

## Classic job scheduling systems - Slurm

- Slurm is an abbreviation for "Simple Linux Utility for Resource Management"
- Slurm has three key features: |
    1. Allocation of exclusive/non-exclusive access to resources
    2. Providing a framework for starting, executing and monitoring tasks on a set of allocated nodes
    3. Managing pending jobs in queues until they can be executed
- Slurm is used on 60% of the TOP500 supercomputers |

+++

### Features

- Modular design (supports plugins)
- Highly scalable |
- Fair-share scheduling (with hierarchical bank accounts) |
- Preemptive and gang scheduling |
- Accounting |
- Different operating systems can be booted for each job |
- Scheduling for generic resources (e.g. GPUs) |
- Resource limits for users/bank account |

Note:
Slurm has a very modular design that supports the installation plugins.
The following list contains only features related to the topic of this talk.

Gang scheduling takes care that multiple related threads are executed on different processors to ensure that all threads are active at the same time to avoid blocking e.g. in producer-consumer systems because the producer is running while the consumer is sleeping.

---

## Comparison Slurm vs. Kubernetes

<table class="default">
    <tr>
        <th class="default">Feature</th>
        <th class="default">Slurm</th>
        <th class="default">Kubernetes</th>
    </tr>
    <tr class="fragment">
        <td class="odd">Highly scalable</td>
        <td class="odd"><i class="fa fa-thumbs-up"></i></td>
        <td class="odd"><i class="fa fa-thumbs-up"></i></td>
    </tr>
    <tr class="fragment">
        <td class="even">Fair scheduling</td>
        <td class="even"><i class="fa fa-thumbs-up"></i></td>
        <td class="even"><i class="fa fa-thumbs-up"></i></td>
    </tr>
    <tr class="fragment">
        <td class="odd">Gang scheduling</td>
        <td class="odd"><i class="fa fa-thumbs-up"></i></td>
        <td class="odd"><i class="fa fa-thumbs-up"></i><br/>
        with<a href="https://github.com/kubernetes-incubator/kube-arbitrator"> kube-arbitrator</a></td>
    </tr>
    <tr class="fragment">
        <td class="even">Accounting</td>
        <td class="even"><i class="fa fa-thumbs-up"></i></td>
        <td class="even"><i class="fa fa-thumbs-down"></i></td>
    </tr>
    <tr class="fragment">
        <td class="odd">Different OS for each job</td>
        <td class="odd"><i class="fa fa-thumbs-up"></i></td>
        <td class="odd"><i class="fa fa-thumbs-up"></i></td>
    </tr>
    <tr class="fragment">
        <td class="even">Scheduling of GPUs</td>
        <td class="even"><i class="fa fa-thumbs-up"></i></td>
        <td class="even"><i class="fa fa-thumbs-up"></i><br/>
        still in alpha state</td>
    </tr>
    <tr class="fragment">
        <td class="odd">Scheduling of generic resources</td>
        <td class="odd"><i class="fa fa-thumbs-up"></i></td>
        <td class="odd"><i class="fa fa-thumbs-down"></i></td>
    </tr>
    <tr class="fragment">
        <td class="even">Resource limits for users/bank account</td>
        <td class="even"><i class="fa fa-thumbs-up"></i></td>
        <td class="even"><i class="fa fa-thumbs-up"></i><br/>
        Resource limits can be set for namespaces</td>
    </tr>
</table>

---

## Kubernetes jobs

<ul>
  <li class="fragment">Jobs are natively supported by Kubernetes</li>
  <li class="fragment">Jobs can be executed in a single <i>pod</i> or in multiple parallel <i>pods</i></li>
  <li class="fragment">A job <b>should always</b> define resource requests and limits</li>
  <li class="fragment">A job is recognized as completed when the container exits with a exit code <code>0</code></li>
</ul>

Note:

- Explain pods
- Explain requests and limits

+++

### Simple sample job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

@[1-4](Required metadata to declare Kubernetes resource)
@[9-11](Declare the container that is running the job)

+++

### Job with specified resource allocation policy

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: john-job
spec:
  template:
    spec:
      containers:
        - name: john
          image: knsit/johntheripper:latest
          command: ["/bin/bash", "-c", "..."]
          resources:
            requests:
              cpu: 1000m
            limits:
              nvidia.com/gpu: 1
              cpu: 2000m
              memory: 4Gi
```

@[8-11](Declare the container that is running the job)
@[13-14](Declare resource requests)
@[16](Job requires one GPU)
@[17-18](Declare resource limits for the job)

---

## Parallel jobs in Kubernetes

Kubernetes offers two different kinds of parallel jobs:

1. Jobs with fixed count of parallel workers
2. Jobs based on a work queue

+++

### Jobs with fixed count of completions

Declare the job like this:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job-1
spec:
  template:
    ...
  completions: 10
```

@[8](Declare how many completions (exit code `0`) are required until the job is finished)

+++

@title[Jobs with fixed count of completions - considerations - part 2]
### Jobs with fixed count of completions - considerations

<ul>
  <li class="fragment">A job that declares a fixed count of completions has a default <code>spec.parallelism</code> value of <code>1</code> but that value can be increased</li>
  <li class="fragment">If a pod fails it might be restarted depending on the values for the <code>backoffLimit</code> and the <code>restartPolicy</code></li>
</ul>

+++

@title[Jobs with fixed count of completions - considerations - part 1]
### Jobs with fixed count of completions - considerations

<ul>
  <li class="fragment">In a future release Kubernetes will pass the partition index (a value between <code>1</code> and <code>spec.completions</code>) to each pod to enable the pod to work only on his partition without the need for any external coordinator</li>
  <li class="fragment">A higher number for `spec.parallelism` than <code>spec.completions</code> is ignored and will fallback to <code>spec.completions</code></li>
</ul>

+++

### Jobs with a work queue

Declare the job like this:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job-1
spec:
  template:
    ...
  parallelism: 10
```

@[8](Specify parallelism count. Kubernetes creates as many pods as specified.)

+++

@title[Jobs with a work queue - considerations - part 1]
### Jobs with a work queue - considerations

<ul>
  <li class="fragment">With a higher parallelism count than <code>1</code> the work can be distributed across multiple nodes</li>
  <li class="fragment">Developers have to take care that work is distributed (approximately) even on all available pods</li>
</ul>

+++

@title[Jobs with a work queue - considerations - part 2]
### Jobs with a work queue - considerations

<ul>
  <li class="fragment">Parallelism count might not be reached if not enough resources are available (<code>requests</code> for CPU, memory or other resources)</li>
  <li class="fragment">Overhead of higher parallelism count is higher than in single <i>pod</i> jobs because <i>pod</i> creation takes its time</li>
</ul>

---

## Conclusion

- It depends on the use case if Kubernetes meets the requirements |
- There are already a lot of success stories where Kubernetes is used as a job scheduler for ML and Big Data tasks (e.g. RiseML, kube-arbitrator) |
- Kubernetes is not a fully fledged job scheduler compared to Slurm (yet) as features like accounting are missing |