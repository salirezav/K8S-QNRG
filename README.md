# Installing and running DC/OS on an On-Premise cluster

This document walks you through the steps needed to:

- Install and run DC/OS on our cluster
- Managing nodes (Adding and removing master/worker nodes)
- Adding and executing services

## Prerequisites

All the machines on our cluster run Ubuntu. Before you start make sure to run `sudo apt-get update` on all of them.  

### System Requirements

A DC/OS cluster consists of two types of nodes **master nodes** and **agent nodes**. The agent nodes can be either **public agent node**s or **private agent nodes**. Public agent nodes provide north-south (external to internal) access to services in the cluster through load balancers. Private agents host the containers and services that are deployed on the cluster. In addition to the master and agent cluster nodes, each DC/OS installation includes a separate **bootstrap node** for DC/OS installation and upgrade files.  
For our cluster I chose **"Ronon"** to be our bootstrap node. on Ronon all installation and upgrade scripts are generated and then provided to other nodes on the cluster to use.
