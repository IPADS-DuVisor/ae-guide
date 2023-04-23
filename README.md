# DuVisor AE Guide

This guide is for artifact evaluation (AE) of paper Security and Performance in the Delegated User-level Virtualization.

The AE of DuVisor contains 2 parts: **Security Evaluation** and **Performance Evaluation**.

<!--ts-->
* [DuVisor AE Guide](#duvisor-ae-guide)
   * [Security Evaluation](#security-evaluation)
   * [Performance Evaluation](#performance-evaluation)
      * [Step 0: Configure AWS Tools](#step-0-configure-aws-tools)
      * [Step 1: Clone Repositories](#step-1-clone-repositories)
      * [Step 2: Setup Tests](#step-2-setup-tests)
      * [Step 3: Automated Testing (Estimated 20 Hours)](#step-3-automated-testing-estimated-20-hours)
      * [Step 4: Generate Figures](#step-4-generate-figures)
<!--te-->

## Security Evaluation

The security AE reproduces table 6 of the paper. Please refer to [security-evaluation.md](./security-evaluation.md)

## Performance Evaluation

The performance AE (the following content of this guide) reproduces figure 7-10 of the paper.

The performance AE is **push-button** style. We provided a configured environment and all pre-built images for simplicity. If you want to build from source, please refer to [build-from-source.md](./build-from-source.md).

The performance AE involves three machines.
* The client machine, which serves as the monitor of the experiments. You operate on this machine to connect with the master machine. Evaluation results would be sent to this machine.
* The master machine, which runs firesim and connects with the worker FPGA.
* The worker machine, which is the FPGA node and experiments run on this FPGA.

In our setup, the client machine is a T3 instance, the master machine is a C5 instance and the worker machine is an F1 instance.
The relation graph of the three machines could be depicted as:

```javascript
Client(Monitor) <====> Master(Firesim) <====> Worker(FPGA)
```

The following is the quick start of the performance AE. All steps should be done on the client machine.

### Step 0: Configure AWS Tools

The AE requires a private AWS account to purchase and launch instances. As a result, we will only provide configured machines for reviewers. For reviewers, please send public keys to us through HotCRP and skip this step.

People can also use their own AWS accounts to create their machines.
Then they need to configure AWS tools on the client machine to connect to the master machine.
This step installs AWS tools and configures with the information of the master machine. The installation commands refer to https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

aws configure
```

### Step 1: Clone Repositories

This step would clone repositories needed for AE.

You should clone two repositories:

* `ae-aws-scripts`. This repository has scripts for connecting to the master machine.
* `ae-firesim-scripts`. This repository has scripts for running tests and pre-built images.

```bash
# Scripts for connecting to master. 
git clone https://github.com/IPADS-DuVisor/ae-aws-scripts.git ~/aws-scripts
# A pem file is required under ~/aws-scripts/west/firesim.pem to access AWS instances.
# We only provide the pem file for reviewers for security reason.

# Test scripts and pre-built images
git clone https://github.com/IPADS-DuVisor/ae-firesim-scripts.git ~/firesim
```

### Step 2: Setup Tests

This step setups the tests for reproducing figure 7-10 of this paper.

In this step, you would create a test configuration file and do some other preparations. 

```bash
# Create /tmp/cmd-seq from template
cp ~/firesim/example.seq /tmp/cmd-seq 
# The *fig* directory is used to put all the figures
mkdir -p fig
```

Explanation of this `example.seq` file is as follow. Each non-empty line represents either an experiment or a comment.

* `#` means line comment.
* The first part is `fig8 app`, which is tests for figure 8. There are two tokens per line.
  * <details><summary>Content</summary><p>
  
      ```txt
      # fig8 app
      kvm 1
      kvm 2
      kvm 4
      native 1
      native 2
      native 4
      ulh 1
      ulh 2
      ulh 4
      ```
  
  </p></details>

  * The first token refers to machine type. `kvm` refers to KVM guest VM, `native` refers to bare-metal machine, and `ulh` refers to DuVisor guest VM.
  * The second token refers to CPU/vCPU number.
* The next part is `fig7 microbenchmark`, which is tests for figure 7. There are two tokens per line.
  * <details><summary>Content</summary><p>
  
      ```txt
      # fig7 microbenchmark
      breakdown hypercall
      breakdown s2pf
      breakdown mmio
      ub hypercall
      ub s2pf
      ub mmio
      breakdown vipi
      vanilla-breakdown vipi
      breakdown vplic
      vanilla-breakdown vplic
      ```
  
  </p></details>

  * The first token refers to machine type. `breakdown` refers to KVM/KVM-opt guest VM, `ub` refers to DuVisor guest VM, and `vanilla-breakdown` refers to KVM guest VM. Note: KVM-opt and DuVisor spend the same cycles on either vipi or vplic tests due to pure-hardware exitless interrupt virtualization.
  * The second token refers to microbenchmark type.
* The next part is `fig9`, which is tests for figure 9. There are four tokens per line.
  * <details><summary>Content</summary><p>
  
      ```txt
      # fig9
      kvm 1 fig9 dv
      kvm 1 fig9 vanilla
      kvm 2 fig9 dv
      kvm 2 fig9 vanilla
      kvm 4 fig9 dv
      kvm 4 fig9 vanilla
      ```
  
  </p></details>

  * The first token is fixed as `kvm` and means this is tested on KVM.
  * The second token refers to vCPU number.
  * The third token is fixed as `fig9`.
  * The fourth token refers to machine type. `dv` refers to KVM-DVext and `vanilla` refers to KVM.
* The next part is `fig10a`, which is tests for figure 10a. There are four tokens per line.
  * <details><summary>Content</summary><p>
  
      ```txt
      # fig10a
      kvm 4 fig10 512
      kvm 4 fig10 1024
      kvm 4 fig10 1536
      kvm 4 fig10 2048

      ulh 4 fig10 512
      ulh 4 fig10 1024
      ulh 4 fig10 1536
      ulh 4 fig10 2048
      ```
  
  </p></details>

  * The first token refers to machine type. `kvm` refers to KVM guest VM, and `ulh` refers to DuVisor guest VM.
  * The second token refers to vCPU number.
  * The third token is fixed as `fig10`.
  * The fourth token refers to memory size with MB unit.
* The last part is `fig10b`, which is tests fot figure 10b. There are five tokens per line.
  * <details><summary>Content</summary><p>
  
      ```txt
      # fig10b
      ulh 1 fig10b pmp cp
      ulh 1 fig10b pmp wr
      ulh 1 fig10b pmp rd
      ulh 1 fig10b pmp rdwr
      ulh 1 fig10b pmp frd
      ulh 1 fig10b pmp fwr
      ulh 1 fig10b pmp fcp
      ulh 1 fig10b pmp bzero
      ulh 1 fig10b pmp bcopy
      ulh 1 fig10b nopmp cp
      ulh 1 fig10b nopmp wr
      ulh 1 fig10b nopmp rd
      ulh 1 fig10b nopmp rdwr
      ulh 1 fig10b nopmp frd
      ulh 1 fig10b nopmp fwr
      ulh 1 fig10b nopmp fcp
      ulh 1 fig10b nopmp bzero
      ulh 1 fig10b nopmp bcopy
      ```
  
  </p></details>

  * The first token is fixed as `ulh` and means this is tests of DuVisor.
  * The second token is fixed as `1` refers to vCPU number.
  * The third token is fixed as `fig10b`.
  * The fourth token refers to hardware type, PMP or no-PMP.
  * The fifth token refers to lmbench type.


### Step 3: Automated Testing (Estimated 20 Hours)

This step runs tests with an integrated push-button script.

`tmux` is recommended in case of `auto-test.sh` interruption due to network failure.

```bash
# tmux new -s duvisor-ae
cd ~/firesim
rm log/*
./auto-test.sh
```

### Step 4: Generate Figures

```bash
cd ~/firesim && mkdir -p ./fig
./data_wraggle.sh
```

After above procedure, you can check stat data by `./show_stat.sh` or check figures in `fig/`.
Since some experiments may need to retry multiple times (e.g. vipi experiments), we recommend you to use `show_csv.py` to watch raw data and use `show_stat.sh` to watch statistics data. If an experiment encounters unexpected errors, please refill the `/tmp/cmd-seq` with this experiment to re-run it.
