@title[Introduction]

# ML and Big Data scheduling in Docker cluster management systems

MaSe - Peter Kurfer

---

## Agenda

- Motivation
- Classic job scheduling systems - Slurm |
- Comparison criteria |

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
- Slurm has three key features:
    1. Allocation of exclusive/non-exclusive access to ressources
    2. Providing a framework for starting, executing and monitoring tasks on a set of allocated nodes
    3. Managing pending jobs in queues until they can be executed
- Slurm is used on 60% of the TOP500 supercomputers

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