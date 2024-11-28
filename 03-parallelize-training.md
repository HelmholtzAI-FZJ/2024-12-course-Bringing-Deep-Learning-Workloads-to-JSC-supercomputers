---
author: Alexandre Strube // Sabrina Benassou // José Ignacio Robledo
title: Bringing Deep Learning Workloads to JSC supercomputers
subtitle: Parallelize Training
date: December 5th, 2024

---

## Before Starting

- We need to download some code

    ```bash
    cd $HOME/course
    git clone https://github.com/HelmholtzAI-FZJ/2024-12-course-Bringing-Deep-Learning-Workloads-to-JSC-supercomputers.git
    ```

- Move to the correct folder
    
    ```bash
    cd code/parallelize
    ```

---

## What this code does 

- It trains a [transformer](https://arxiv.org/pdf/1706.03762) model on the [xsum](https://paperswithcode.com/dataset/xsum) dataset to summarize documents.
- Again, this is not a deep learning course.
- If you are not familiar with the model and the dataset, just imagine it as a black box: you provide it with text, and it returns a summary.

---

Let's have a look at the files **```train.py```** and **```run_train.sbatch```** in the repo.

![](images/look.jpg)

---

## Run the Training Script

- Now run:

    ```bash
    sbatch run_train.sbatch 
    ```

- Spoiler alert 🚨

- The code won't work.

- Check the output and error files

---

## What is the problem?

- Remember, there is no internet on the compute node.
- Therefore, you should:
    - Comment lines 90 to 130.
    - Activate your environment:

        ```bash
        source $HOME/course/$USER/sc_venv_template/activate.sh
        ```

    - Run:

        ```bash
        python train.py
        ```

    - Uncomment lines 90-130.
    - Finally, run your job again 🚀:

        ```bash
        sbatch run_train.sbatch
        ```


--- 

## What's about many gpus ?  👀

- Congrats, you are training a DL model on the supercomputer using one GPU 🎉

- Can we run our model on multiple GPUs ?

---

## What if

- In file **```run_train.sbatch```**, we increase the number of GPUs at line 3 to 4:

    ```bash
    #SBATCH --gres=gpu:4
    ```

- And we change the values of the variable ```CUDA_VISIBLE_DEVICES``` at line 13 as follow:

    ```bash
    export CUDA_VISIBLE_DEVICES=0,1,2,3
    ```

--- 

## IT WON't work

- We don't have an established communication between the GPUs
- So, each GPU will perform its training independently.

    ![](images/dist/no_comm.svg){height=450px}

---

## We need communication

![](images/dist/comm1.svg){height=500px}

---

## We need communication

![](images/dist/comm2.svg){height=500px}

---

## collective operations

- The GPUs use collective operations to communicate and share data in parallel computing
- The most common collective operations are: All Reduce, All Gather, and Reduce Scatter

---

## All Reduce 

![](images/dist/all_reduce.svg)

- Other operations, such as **min**, **max**, and **avg**, can also be performed using All-Reduce.

---

## All Gather

![](images/dist/all_gather.svg)

--- 

## Reduce Scatter

![](images/dist/reduce_scatter.svg)

--- 

## Terminologies

- Before going further, we need to learn some terminologies

---

## World Size

![](images/dist/gpus.svg){height=550px}

---

## Rank

![](images/dist/rank.svg){height=550px}

---

## local_rank

![](images/dist/local_rank.svg){height=550px}

---

## Now

That we have understood how the devices communicate and the terminologies used in parallel computing, 
we can move on to distributed training (training on multiple GPUs).

---

## Distributed Training

- Parallelize the training across multiple nodes, 
- Significantly enhancing training speed and model accuracy.
- It is particularly beneficial for large models and computationally intensive tasks, such as deep learning.[[1]](https://pytorch.org/tutorials/distributed/home.html)


---

## Distributed Data Parallel (DDP)

[DDP](https://pytorch.org/tutorials/intermediate/ddp_tutorial.html) is a method in parallel computing used to train deep learning models across multiple GPUs or nodes efficiently.

![](images/ddp/ddp-2.svg){height=400px}

--- 

## DDP

![](images/ddp/ddp-3.svg){height=500px}

--- 

## DDP

![](images/ddp/ddp-4.svg){height=500px}

--- 

## DDP

![](images/ddp/ddp-5.svg){height=500px}

--- 

## DDP

![](images/ddp/ddp-6.svg){height=500px}

--- 

## DDP

![](images/ddp/ddp-7.svg){height=500px}

--- 

## DDP

![](images/ddp/ddp-8.svg){height=500px}

--- 

## DDP

![](images/ddp/ddp-9.svg){height=500px}

--- 

## DDP recap

- Each GPU on each node gets its own process.
- Each GPU has a copy of the model.
- Each GPU has visibility into a subset of the overall dataset and will only see that subset.
- Each process performs a full forward and backward pass in parallel and calculates its gradients.
- The gradients are synchronized and averaged across all processes.
- Each process updates its optimizer.

---

## Let's start coding!

- Whenever you see **TODOs**💻📝, it means you need to follow the instructions to either copy-paste the code or type it yourself.

---

## Setup communication

- We need to setup a communication among the GPUs. 
- For that we would need the file **```distributed_utils.py```**.
- **TODOs**💻📝:
    1. Import **```distributed_utils```** file at line 13:
        
        ```python 
        # This file contains utility_functions for distributed training.
        from distributed_utils import *
        ```
    2. Then **remove** lines 77 and 78:

        ```python
        device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        ```
    3. and **add** at line 77 a call to the method **```setup()```** defined in **```distributed_utils.py```**: 

        ```python
        # Initialize a communication group and return the right identifiers.
        local_rank, rank, device = setup()
        ```

---

## Setup communication

What is in the **```setup()```** method ?

```python
def setup():
    # Initializes a communication group using 'nccl' as the backend for GPU communication.
    torch.distributed.init_process_group(backend='nccl')
    # Get the identifier of each process within a node
    local_rank = int(os.getenv('LOCAL_RANK'))
    # Get the global identifier of each process within the distributed system
    rank = int(os.environ['RANK'])
    # Creates a torch.device object that represents the GPU to be used by this process.
    device = torch.device('cuda', local_rank)
    # Sets the default CUDA device for the current process, 
    # ensuring all subsequent CUDA operations are performed on the specified GPU device.
    torch.cuda.set_device(device)
    # Different random seed for each process.
    torch.random.manual_seed(1000 + torch.distributed.get_rank())

    return local_rank, rank, device
```

---

## Model

- **TODO 4**💻📝:

    - At line 83, wrap the model in a **DistributedDataParallel** (DDP) module to parallelize the training across multiple GPUs.
    
        ```python 
        # Wrap the model in DistributedDataParallel module 
        model = torch.nn.parallel.DistributedDataParallel(
            model,
            device_ids=[local_rank],
        )
        ```

---

## DistributedSampler 

- **TODO 5**💻📝:

    - At line 94, instantiate a **DistributedSampler** object for each set to ensure that each process gets a different subset of the data.
    
        ```python
        # DistributedSampler object for each set to ensure that each process gets a different subset of the data.
        train_sampler = torch.utils.data.distributed.DistributedSampler(train_dataset, 
                                                                        shuffle=True, 
                                                                        seed=args.seed)
        val_sampler = torch.utils.data.distributed.DistributedSampler(val_dataset)
        test_sampler = torch.utils.data.distributed.DistributedSampler(test_dataset)
        ```

---

## DataLoader

- **TODO 6**💻📝:

    - At line 103, **REMOVE** **```shuffle=True```** in the DataLoader of train_loader and **REPLACE** it by **```sampler=train_sampler```**
        
        ```python 
        train_loader = DataLoader(train_dataset, 
                                batch_size=args.batch_size, 
                                sampler=train_sampler, # pass the sampler argument to the DataLoader
                                num_workers=int(os.getenv('SLURM_CPUS_PER_TASK')),
                                pin_memory=True)
        ```

---

## DataLoader

- **TODO 7**💻📝:

    -  At line 108, pass **val_sampler** to the sampler argument of the val_dataLoader

        ```python
        val_loader = DataLoader(val_dataset,
                                batch_size=args.batch_size,
                                sampler=val_sampler, # pass the sampler argument to the DataLoader
                                pin_memory=True)
        ```

- **TODO 8**💻📝:

    - At line 112, pass **test_sampler** to the sampler argument of the test_dataLoader

        ```python
        test_loader = DataLoader(test_dataset,
                                batch_size=args.test_batch_size,
                                sampler=test_sampler, # pass the sampler argument to the DataLoader
                                pin_memory=True)    
        ```

--- 

## Sampler 

- **TODO 9**💻📝:

    - At line 125, **set** the current epoch for the dataset sampler to ensure proper data shuffling in each epoch

        ```python
        # Pass the current epoch to the sampler to ensure proper data shuffling in each epoch
        train_sampler.set_epoch(epoch)
        ```

---

## All Reduce Operation

- **TODO 10**💻📝:

    - At **lines 49 and 72**, Obtain the global average loss across the GPUs.

        ```python
        # Return the global average loss.
        torch.distributed.all_reduce(result, torch.distributed.ReduceOp.AVG)
        ```

---

## print

- **TODO 11**💻📝:

    - At **lines 133, 144, and 148**, **replace** all the ```print``` methods by **```print0```** method defined in **```distributed_utils.py```** to allow only rank 0 to print in the output file.
    
        ```python
        # We use the utility function print0 to print messages only from rank 0.
        print0(f'[{epoch+1}/{args.epochs}; {i}] Train loss: {train_loss:.5f}, validation loss: {val_loss:.5f}')
        ```
        ```python
        # We use the utility function print0 to print messages only from rank 0.
        print0('Finished training after', end_time - start_time, 'seconds.')
        ```
        ```python
        # We use the utility function print0 to print messages only from rank 0.
        print0('Final test loss:', test_loss.item())
        ```

---

## print

The definition of the function **print0** is in **```distributed_utils.py```**

```python
functools.lru_cache(maxsize=None)
def is_root_process():
    """Return whether this process is the root process."""
    return torch.distributed.get_rank() == 0


def print0(*args, **kwargs):
    """Print something only on the root process."""
    if is_root_process():
        print(*args, **kwargs)
```

---

## Save model 

- **TODO 12**💻📝:

    - At **lines 138 and 151**, replace torch.save method with the utility function save0 to allow only the process with rank 0 to save the model.
 
        ```python 
        # We allow only rank=0 to save the model
        save0(model, 'model-best')
        ```
        ```python 
        # We allow only rank=0 to save the model
        save0(model, 'model-final')
        ```

---

## Save model 

The method **save0** is defined in **```distributed_utils.py```**

```python
functools.lru_cache(maxsize=None)
def is_root_process():
    """Return whether this process is the root process."""
    return torch.distributed.get_rank() == 0


def save0(*args, **kwargs):
    """Pass the given arguments to `torch.save`, but only on the root
    process.
    """
    # We do *not* want to write to the same location with multiple
    # processes at the same time.
    if is_root_process():
        torch.save(*args, **kwargs)
```

--- 

## We are almost there

- That's it for the **train.py** file. 
- But before launching our job, we need to add some lines to **run_train.sbatch** file 

---

## Setup communication

In **```run_train.sbatch```** file:

- **TODOs 13**💻📝: **if it is not already done**
    - At line 3, increase the number of GPUs to 4.

        ```bash
        #SBATCH --gres=gpu:4
        ```

    - At line 14, pass the correct number of devices.

        ```bash
        export CUDA_VISIBLE_DEVICES=0,1,2,3
        ```

---

## Setup communication

Stay in **```run_train.sbatch```** file:

- **TODO 14**💻📝: we need to setup **MASTER_ADDR** and **MASTER_PORT** to allow communication over the system.

    - At line 16, add the following:

        ```bash
        # Extracts the first hostname from the list of allocated nodes to use as the master address.
        MASTER_ADDR="$(scontrol show hostnames "$SLURM_JOB_NODELIST" | head -n 1)"
        # Modifies the master address to allow communication over InfiniBand cells.
        MASTER_ADDR="${MASTER_ADDR}i"
        # Get IP for hostname.
        export MASTER_ADDR="$(nslookup "$MASTER_ADDR" | grep -oP '(?<=Address: ).*')"
        export MASTER_PORT=7010
        ```

---

## Setup communication

We are not done yet with **```run_train.sbatch```** file:

- **TODO 15**💻📝: 
    
    - At line 27, we change the lauching script to use **torchrun_jsc** and pass the following argument: 

        ```bash
        # Launch a distributed training job across multiple nodes and GPUs
        srun --cpu_bind=none bash -c "torchrun \
            --nnodes=$SLURM_NNODES \
            --rdzv_backend c10d \
            --nproc_per_node=gpu \
            --rdzv_id $RANDOM \
            --rdzv_endpoint=$MASTER_ADDR:$MASTER_PORT \
            --rdzv_conf=is_host=\$(if ((SLURM_NODEID)); then echo 0; else echo 1; fi) \
            train.py "
        ```

---

## Setup communication

- The arguments that we pass are:

    1. **```nnodes=$SLURM_NNODES```**: the number of nodes
    2. **```rdzv_backend c10d```**: the c10d method for coordinating the setup of communication among distributed processes.
    3. **```nproc_per_node=gpu```** the number of GPUs
    4. **```rdzv_id $RANDOM```** a random id which that acts as a central point for initializing and coordinating the communication among different nodes participating in the distributed training. 
    5. **```rdzv_endpoint=$MASTER_ADDR:$MASTER_PORT```** the IP that we setup in the previous slide to ensure all nodes know where to connect to start the training session.
    6. **```rdzv_conf=is_host=\$(if ((SLURM_NODEID)); then echo 0; else echo 1; fi)```** The rendezvous host which is responsible for coordinating the initial setup of communication among the nodes.

---

## done ✅

- You can finally run:

    ```bash
    sbatch run_train.sbatch
    ```

---

## Congrats 👏

- You have run your model on 4 GPUs 

- But what about many nodes ?

---

## Multi-node training

- In **```run_train.sbatch```** at line 2, you can increase the number of nodes to 2:

    ```bash
    #SBATCH --nodes=2
    ```

- Hence, you will use 8 GPUs for training.

- Run again:

    ```bash
    sbatch run_train.sbatch
    ```

--- 

## That's it for DDP

---

## Before we go further...

- Data parallel is usually good enough 👌 
- However, if your model is too big to fit into a single GPU
- Welllll ... there other distributed techniques ...

---

## Fully Sharded Data Parallel (FSDP)

![](images/fsdp/fsdp-0.svg)

---

## FSDP

![](images/fsdp/fsdp-1.svg)


---

## FSDP

![](images/fsdp/fsdp-2.svg)


---

## FSDP

![](images/fsdp/fsdp-3.svg)


---

## FSDP

![](images/fsdp/fsdp-4.svg)


---

## FSDP

![](images/fsdp/fsdp-5.svg)


---

## FSDP

![](images/fsdp/fsdp-6.svg)


---

## FSDP

![](images/fsdp/fsdp-7.svg)


---

## FSDP

![](images/fsdp/fsdp-8.svg)


---

## FSDP

![](images/fsdp/fsdp-9.svg)


---

## FSDP

![](images/fsdp/fsdp-10.svg)


---

## FSDP

![](images/fsdp/fsdp-11.svg)


---

## FSDP

![](images/fsdp/fsdp-12.svg)


---

## FSDP

![](images/fsdp/fsdp-13.svg)


---

## FSDP

![](images/fsdp/fsdp-14.svg)


---

## FSDP

![](images/fsdp/fsdp-15.svg)


---

## FSDP

![](images/fsdp/fsdp-16.svg)


---

## FSDP

![](images/fsdp/fsdp-17.svg)


---

## FSDP

![](images/fsdp/fsdp-18.svg)

---

## FSDP

![](images/fsdp/fsdp-19.svg)

---

## FSDP

![](images/fsdp/fsdp-20.svg)


---

## FSDP

- FSDP is again good enough
- It is a primitive method to PyTorch 
- However, it require a high-bandwidth system
- If you have bandwidth-limited clusters FSDP maybe not good and would prefer Pipelining

---

## Model Parallel

- Model *itself* is too big to fit in one single GPU 🐋
- Each GPU holds a slice of the model 🍕
- Data moves from one GPU to the next

---

## Model Parallel

![](images/model-parallel.svg)

---


## Model Parallel

![](images/model-parallel-pipeline-1.svg)

---

## Model Parallel

![](images/model-parallel-pipeline-2.svg)

---

## Model Parallel

![](images/model-parallel-pipeline-3.svg)

---

## Model Parallel

![](images/model-parallel-pipeline-4.svg)

---

## Model Parallel

![](images/model-parallel-pipeline-5.svg)

---

## Model Parallel

![](images/model-parallel-pipeline-6.svg)

---

## Model Parallel

![](images/model-parallel-pipeline-7.svg)

---

## Model Parallel

![](images/model-parallel-pipeline-8.svg)

---

## Model Parallel

![](images/model-parallel-pipeline-9.svg)

---

## Model Parallel

![](images/model-parallel-pipeline-10.svg)

---

## What's the problem here? 🧐

---

## Model Parallel

- Waste of resources
- While one GPU is working, others are waiting the whole process to end
- ![](images/no_pipe.png)
    - [Source: GPipe: Efficient Training of Giant Neural Networks using Pipeline Parallelism](https://arxiv.org/abs/1811.06965)


---

## Model Parallel - Pipelining

![](images/model-parallel-pipeline-1.svg)

---

## Model Parallel - Pipelining

![](images/model-parallel-pipeline-2-multibatch.svg)

---

## Model Parallel - Pipelining

![](images/model-parallel-pipeline-3-multibatch.svg)

---

## Model Parallel - Pipelining

![](images/model-parallel-pipeline-4-multibatch.svg)

---

## Model Parallel - Pipelining

![](images/model-parallel-pipeline-5-multibatch.svg)

---

## Model Parallel - Pipelining

![](images/model-parallel-pipeline-6-multibatch.svg)

---

## Model Parallel - Pipelining

![](images/model-parallel-pipeline-7-multibatch.svg)

---

## Model Parallel - Pipelining

![](images/model-parallel-pipeline-8-multibatch.svg)

---

## Model Parallel - Pipelining

![](images/model-parallel-pipeline-9-multibatch.svg)

---

## This is an oversimplification!

- Actually, you split the input minibatch into multiple microbatches.
- There's still idle time - an unavoidable "bubble" 🫧
- ![](images/pipe.png)

---

## Model Parallel - Multi Node

- In this case, each node does the same as the others. 
- At each step, they all synchronize their weights.

---

## Model Parallel - Multi Node

![](images/model-parallel-multi-node.svg)

---

---

## Llview
- [llview](https://go.fzj.de/llview-jureca)
- https://go.fzj.de/llview-jureca
![](images/llview.png)

---

## Part 2 RECAP 

- You know what is a distributed training.
- Can submit single GPU, multi-GPU and multi-node training using DDP.
- You know other distributed training techniques.
- Usage of llview.

---

## ANY QUESTIONS??

#### Feedback is more than welcome!

#### Link to [other courses at JSC](https://go.fzj.de/dl-in-neuroscience-all-courses)

---

