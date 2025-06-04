# Introduction
The batch scheduler system is typically a crucial component to any modern supercomputer. When a user first connects to an HPC system, they will typically be placed onto a *login (or front-end) node*. These login nodes are similar in form and function to a Linux workstation, though they usually lack a graphical user interface. Login nodes are suitable for basic tasks such as text editing, scripting, data interrogation, and short, low-resource processing. However, to gain access to the vast quantity of resources available on a supercomputer, the user will need to assign work to one or more *compute (or back-end) nodes*. The batch scheduler provides the mechanism for users to perform such assignments.

Multiple batch schedulers exist in the current HPC landscape, including Slurm, PBS Pro, and LSF. Some systems are even repurposing cloud-orchestration systems like Kubernetes to assign work on large clusters. However, not all schedulers offer the same functionality.

At NSF NCAR, we currently provide two large clusters to our user community {cite}`ncarhpc`. Each system has a defined purpose, and both rely on the PBS Pro batch scheduler to assign work.

* **Derecho** - our HPC system intended for large modeling experiments and other highly-scalable work like machine learning
* **Casper** - our data analysis, visualization, and heterogeneous workflow machine

For a time, Casper ran the Slurm scheduler, and so our users and support staff have had experience with both PBS Pro and Slurm in recent years. While each scheduler does have its strengths, we have found that Slurm provides convenient interfaces to useful information that are either not readily available or have limitations in PBS Pro. Therefore, we have developed the following additional tools which are deployed on both of our systems:

| NCAR PBS tool | Slurm equivalent | Motivation |
|-|-|-|
| `qhist` | `sacct` | PBS did not provide a tool out of the box for querying historical job records |
| `qstat-cache` | `squeue` | PBS' native qstat command can impact scheduler performance if load is too high |
| `pbsprio` | `sprio` | PBS did not provide a tool to query factors affecting relative job priority |
| `launch_cf` | `srun`* | User wish to launch many serial tasks easily on the compute nodes |

**`srun` is not a perfect analog for launch_cf, which facilitates MPMD submission of work, but it can be used to accomplish the same  outcome*

### Modernizing the tool set

These tools have seen widespread use by our support staff and/or our user community, but they were written as bespoke solutions to address an immediate need. To ensure the tools are maintainable as languages, staff, and clusters evolve, it is increasingly crucial to use modern software engineering practices.

In this notebook, we undertake a modernization effort for our `qhist` script ([source code repo](https://github.com/NCAR/qhist)), which is written in Python. This user-facing utility allows our community to query the historical records of completed PBS jobs, along with useful job metadata such as CPU utilization, memory usage, and core-hours consumed. While `qhist` provides significant added functionality to our users, the current implementation of this script has some unique shortcomings that are worth highlighting:

* All data are read into memory, and so large queries over long time windows quickly get expensive and may exceed the memory available to the user
* The source records are only available on login nodes (via rsync from the PBS server itself), so `qhist` cannot be used from within compute jobs

These two issues compound each other as well - large queries require memory amounts only available on compute nodes, but those nodes do not provide the required source data.
 
Resolving this quandary was a top priority in the modernization effort. Along the way, a significant modernization of the code infrastructure was performed, utilizing the latest best practices for Python packaging, and testing and deployment via GitHub workflows. Specifically, the following improvements were implemented:

* split out PBS record processing into a separate package, allowing us to reuse code in other use cases;
* improve output formatting and filtering capabilities;
* convert the script into a *package* and make it installable via `pip`;
* provide an alternate `Makefile` installation method for use outside of a Python site library;
* use GitHub Actions {cite}`ghactions` to automate package deployment to [PyPI](https://pypi.org/);
* augment user documentation via a repository `README.md` and a man page;
* add a regression test suite;
* use self-hosted Actions runners to perform live testing on our HPC systems.