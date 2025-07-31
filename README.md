# Improving the PBS Pro Experience for NCAR HPC Users
[![JupyterBook](https://github.com/UCAR-SEA/SEA-ISS-Template/actions/workflows/deploy.yml/badge.svg)](https://github.com/UCAR-SEA/SEA-ISS-Template/actions/workflows/deploy.yml)
[![Made withJupyter](https://img.shields.io/badge/Made%20with-Jupyter-green?style=flat-square&logo=Jupyter&color=green)](https://jupyter.org/try)
![Static Badge](https://img.shields.io/badge/DOI-10.XXXXX%2Fnnnnn-blue)

**Author**: Brian Vanderwende

**Abstract**: For most HPC users, the jobs scheduling software is an integral component of the system, allowing access to the vast compute resources that distinguish a cluster from a workstation. A few workload managers are common in traditional scientific HPC (e.g., Slurm and PBS) with newer tools like Kubernetes also becoming more common. HPC users and administrators are often forced to adapt to a new scheduler upon procurement of the latest system, at which point they experience the strengths and limitations of the new tool.

At NSF NCAR, our two main clusters - Derecho and Casper - both run PBS Pro, though we have used Slurm in the recent past as well. Compared to some of its competitors, PBS Pro's user interface has notable deficiencies: users cannot query historical jobs beyond a few days, administrators cannot query relative job execution priorities, and some queries impose a serious performance impact on the PBS server. To mitigate these weak points, we have developed a number of tools including a cached qstat (job query tool), qhist (a historical record tool), and pbs_prio (a priority query tool).

In this notebook, we focus on the `qhist` utility as a case study, demonstrating recent efforts to modernize our PBS tooling. Such efforts include adding requested features, incorporating more modern Python programming practices, reducing program overhead, improving documentation, converting scripts into actual Python packages, using automated deployment, and adding regression testing.

**Keywords:** modernization, automation, packaging, Python, PBS, performance

**Acknowledgements**: The author would like to acknowledge testing and feature suggestions from colleagues in the HPC Consulting Services Group at NSF NCAR, as well as constructive feedback and bug reports from the NSF NCAR HPC user community as a whole.
