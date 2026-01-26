# Infrastructure Charts

## Overview

This repository contains infrastructure Helm charts for demo projects created in this github org, including cert-manager, HashiCorp Vault, External Secrets Operator, Stakater Reloader (for dynamic configuration reloading), and other essential infrastructure components.

### Prerequisites

Before using these infrastructure charts, ensure you have the following:

- **OpenShift Cluster**
  - **Local Development**: Use Red Hat's CodeReady Containers (CRC) for a local OpenShift cluster. Setup instructions are provided below in the "Local OpenShift Cluster Setup" section.
  - **Cloud/Managed Environment**: Alternatively, use a managed OpenShift service from cloud providers such as Red Hat OpenShift on AWS (ROSA), Azure Red Hat OpenShift (ARO), or OpenShift on GCP. Refer to the official documentation for cloud-specific setup.
  - **Kubernetes Alternative**: While these charts are designed and tested for OpenShift, they can also run on standard Kubernetes clusters (such as those provided by GKE, EKS, or AKS) or a local cluster like Kind, Minikube, etc. But note that minor configuration adjustments may be required for Kubernetes deployments.

- **Helm**: Package manager for Kubernetes/OpenShift (v3.x or later recommended)

- **CLI Tools**: `kubectl` or `oc` (OpenShift CLI) for cluster management and operations

## Local OpenShift Cluster Setup - CodeReady Container

- Prerequisites: First, ensure your system meets the requirements:
  - RAM: Minimum 9 GB (16 GB recommended)
  - CPU: 4 physical CPU cores
  - Disk space: 35 GB free space
  - Operating System: Windows, macOS, or Linux

- Download CRC
  - Visit the Red Hat OpenShift download page and download CRC for your platform.
  - Webpage: <https://developers.redhat.com/content-gateway/rest/mirror/pub/openshift-v4/clients/crc/latest/>
  - Optionally for macos you if you know the version you want you can use: <https://developers.redhat.com/content-gateway/file/pub/openshift-v4/clients/crc/2.57.0/crc-macos-installer.pkg>

    ```shell
    # After installation you can run
    crc version
    # Example Output-1:
    CRC version: 2.57.0+ae41f6
    OpenShift version: 4.20.5
    MicroShift version: 4.20.0
     
    # NOTE: If there is a newer version this command will give us the newer version URL and you can update it by re-installing the new package. 
    # Example Output - 2:
    WARN A new version (2.57.0) has been published on https://developers.redhat.com/content-gateway/file/pub/openshift-v4/clients/crc/2.57.0/crc-macos-installer.pkg
    CRC version: 2.51.0+80aa80
    OpenShift version: 4.18.2
    MicroShift version: 4.18.2
    ```

  - Create a new local OpenShift cluster by running:
    > [!NOTE]
    > Login and download the `pull-secret.txt` file from <https://console.redhat.com/openshift/create/local>

    ```shell
    # One-time setup (first time or after upgrade)
    crc setup

    # Download pull secret from console.redhat.com and save it or you can also provide this file path later when `crc start` command prompts for it.
    crc config set pull-secret-file /path/to/pull-secret.txt

    # Option Resource Configs
    ## Set custom CPU cores (default is 4)
    crc config set cpus 10

    ## Set custom memory in MB (default is 10752)
    crc config set memory 49152

    ## Set disk size in GB (default is 31)
    crc config set disk-size 120

    ## Set user defined kubeadmin password 
    crc config set kubeadmin-password Admin@123 #Note, change this example password.

    ## Set user defined developer password
    crc config set developer-password Dev@123 #Note, change this example password.

    ## Verify or to set other properties
    crc config view
    crc config --help


    # Start existing or create new cluster:
    crc start

    # Set up the oc command in your shell
    eval $(crc oc-env)
    ```

- Use the following command to manage the code ready container - OpenShift cluster for local development.

  ```shell
  ###############################
  # Useful Management Commands
  ###############################
  ## Check cluster status
  crc status

  ## Stop the cluster
  crc stop

  ## Delete the cluster
  crc delete

  ## Start existing or create new cluster:
  crc start

  # Set up the oc command in your shell
  eval $(crc oc-env)

  ## View cluster info
  crc console --credentials

  # Open the console in browser
  crc console

  ## View current configuration
  crc config view

  # Login as admin
  oc login -u kubeadmin https://api.crc.testing:6443
  # Enter the password displayed after 'crc start' or `crc console --credentials`

  # Or login as developer
  oc login -u developer https://api.crc.testing:6443
  ```
  