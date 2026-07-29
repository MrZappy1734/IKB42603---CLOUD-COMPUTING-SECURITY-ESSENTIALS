# Lab 0: Environment Setup Report and Progress Proof

## Course Information

| Item | Details |
| --- | --- |
| Course | IKB42603 Cloud Computing Security Essentials |
| Lab | Lab 0 - Environment Setup |
| Student | Muhammad Hafizi Hasdi |
| Evidence dates | 27 July 2026 and 29 July 2026 |
| Reference | [IKB42603 Lab 0 Environment Setup Cheatsheet](<IKB42603_Lab0_Environment_Setup_Cheatsheet.pdf>) |

## Objective

The purpose of Lab 0 is to prepare a repeatable local environment for the cloud-security exercises in the later labs. The environment needs to run containerised services, simulate selected AWS services locally, and provide a local Kubernetes cluster for deployment and security testing.

For this setup, Docker provides the container runtime. LocalStack is run inside Docker to emulate AWS services without using a real AWS account. The AWS CLI is configured with dummy values and directed to LocalStack, so commands can be tested safely on the local machine. `kind` (Kubernetes IN Docker) creates a Kubernetes cluster using Docker containers, while `kubectl` is used to inspect and manage that cluster. Finally, `oathtool` is available for generating or working with time-based one-time passwords in later security activities.

The following sections explain the commands entered, the reason for each command, and the outcome shown in the screenshots.

## Environment Summary

The evidence was collected from a Kali Linux virtual machine running in Oracle VirtualBox.

| Component | Verified status | Evidence |
| --- | --- | --- |
| Docker | Able to create, run, stop, start, remove, and list containers | Screenshots 2, 3, 5, and 7 |
| LocalStack | Started successfully and responded on port `4566` | Screenshots 2 and 7 |
| AWS CLI configuration | Dummy access key, secret key, and `us-east-1` region configured | Screenshot 5 |
| AWS CLI with LocalStack | `sts get-caller-identity` returned the LocalStack test identity | Screenshot 7 |
| kind | Installed, version `0.23.0`; cluster `ccse` created and deleted successfully | Screenshots 1, 4, and 5 |
| kubectl | Installed, client version `v1.33.4`; Kubernetes node reported as `Ready` | Screenshots 1 and 4 |
| oathtool | OATH Toolkit version `2.6.14` installed | Screenshot 6 |

> Note: The screenshots demonstrate the components listed above. Version checks for Docker, AWS CLI, and OpenSSL are not included in the supplied evidence, so they are not claimed as separate version-verification results in this report.

## Step 1: Verify Kubernetes Tools

The first commands checked that both Kubernetes command-line tools were available:

```bash
kind --version
kubectl version --client
```

`kind --version` confirms that the `kind` executable can be found through the system `PATH` and reports the installed version. This is important because `kind` will later use Docker to build the local Kubernetes cluster. The result shows `kind version 0.23.0`.

`kubectl version --client` checks the client program only; it does not require a Kubernetes cluster to be running at that moment. This command was used to confirm that the tool needed to communicate with and inspect the cluster is installed correctly. The output shows kubectl client version `v1.33.4` and Kustomize version `v5.5.0`.

![Terminal showing kind and kubectl version checks](<Screenshot 2026-07-27 191209.png>)

## Step 2: Start and Check LocalStack

LocalStack was started as a detached Docker container with the following command:

```bash
docker run -d --name localstack -p 4566:4566 localstack/localstack:3
```

`docker run` creates a container from the specified LocalStack image. The `-d` option runs it in the background so the terminal remains available. `--name localstack` gives the container a predictable name, making later Docker commands easier to read and use. The port mapping `-p 4566:4566` exposes LocalStack's main gateway port on the local machine; AWS CLI requests can therefore be sent to `http://localhost:4566`.

The command below was then used to confirm that Docker had started the container:

```bash
docker ps
```

`docker ps` lists currently running containers. The output shows the `localstack/localstack:3` container, its port mapping, and an `Up` status. The LocalStack health URL was also requested:

```bash
curl http://localhost:4566/_localstack/health
```

`curl` sends an HTTP request to the LocalStack health endpoint. The returned JSON lists services such as S3, IAM, Lambda, DynamoDB, and STS as available, confirming that LocalStack is ready to accept local AWS-style requests.

![Terminal showing LocalStack container and health endpoint](<Screenshot 2026-07-27 194101.png>)

## Step 3: Test LocalStack Container Lifecycle Commands

The LocalStack container lifecycle was tested with these Docker commands:

```bash
docker stop localstack
docker start localstack
docker rm -f localstack
```

`docker stop localstack` requests a clean shutdown of the named LocalStack container. This is useful when the service needs to be paused without deleting its container configuration. `docker start localstack` starts an existing stopped container again, which verifies that the named container can be reused. Finally, `docker rm -f localstack` force-removes the container. Removing it is useful when a clean LocalStack instance is needed or when port `4566` is occupied by an old container.

The screenshot shows each command returning the container name `localstack`, which confirms that the stop, restart, and removal operations completed.

![Terminal showing LocalStack stop, start, and removal](<Screenshot 2026-07-27 194519.png>)

## Step 4: Create and Verify the Local Kubernetes Cluster

The local Kubernetes cluster was created with:

```bash
kind create cluster --name ccse
```

This command tells `kind` to create a cluster named `ccse`. Naming the cluster makes its Kubernetes context predictable (`kind-ccse`) and prevents confusion if more than one local cluster exists. Behind the scenes, kind uses Docker to create the control-plane node container and install the core Kubernetes components.

Two commands were used to verify that the cluster was usable:

```bash
kubectl cluster-info --context kind-ccse
kubectl get nodes
```

`kubectl cluster-info --context kind-ccse` queries the cluster using the context created by kind. It confirms that the Kubernetes control plane is reachable and identifies the CoreDNS service. `kubectl get nodes` lists the nodes known to the cluster. The `ccse-control-plane` node is shown with status `Ready`, demonstrating that the cluster finished starting and is ready for later lab workloads.

![Terminal showing creation and verification of the ccse Kind cluster](<Screenshot 2026-07-27 195148.png>)

## Step 5: Remove the Cluster and Configure AWS CLI for LocalStack

After verifying the Kubernetes cluster, it was removed with:

```bash
kind delete cluster --name ccse
```

This command deletes the Docker containers and Kubernetes configuration associated with the cluster named `ccse`. It is a useful cleanup step because it confirms that the cluster can be removed cleanly and avoids leaving unnecessary containers running when the cluster is not needed.

The AWS CLI was then prepared for local testing:

```bash
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
aws configure list
```

The first two commands save dummy credentials named `test`. LocalStack accepts these values, so real AWS credentials are neither required nor appropriate for this lab. The region is set to `us-east-1`, which provides a consistent default region for AWS CLI commands. `aws configure list` displays the active configuration and was used to confirm that the values were stored in the AWS shared credentials and configuration files.

![Terminal showing Kind cluster deletion and AWS CLI configuration](<Screenshot 2026-07-27 203457.png>)

## Step 6: Verify oathtool

The helper tool was checked using:

```bash
oathtool --version
```

`oathtool` is part of the OATH Toolkit. It is used to work with one-time passwords, including TOTP-based multi-factor authentication codes, which are relevant to later security exercises. The `--version` option verifies that the executable is installed and accessible without generating a code or requiring a secret. The output confirms OATH Toolkit version `2.6.14`.

![Terminal showing oathtool version check](<Screenshot 2026-07-27 203525.png>)

## Step 7: Connect AWS CLI to LocalStack

The final check confirmed that AWS CLI requests were routed to LocalStack rather than to the public AWS service. First, the endpoint option was stored in a shell variable:

```bash
EP='--endpoint-url=http://localhost:4566'
```

The variable avoids repeating the full LocalStack URL for every AWS CLI command. It is intended for the current terminal session and can be used by placing `$EP` after `aws`.

The first identity request failed because no LocalStack container was running:

```bash
aws $EP sts get-caller-identity
```

The error, `Could not connect to the endpoint URL`, correctly indicated that nothing was listening on port `4566`. The issue was resolved by starting a new LocalStack container and checking its health endpoint again:

```bash
docker run -d --name localstack -p 4566:4566 localstack/localstack:3
curl http://localhost:4566/_localstack/health
```

After LocalStack responded successfully, the STS identity command was repeated:

```bash
aws $EP sts get-caller-identity
```

`sts get-caller-identity` is a simple AWS CLI test because it returns the identity used for the request. The response contains LocalStack's example user ID, an all-zero account number, and a local IAM root ARN. This confirms both that the AWS CLI command succeeded and that it contacted LocalStack instead of real AWS.

![Terminal showing successful AWS CLI connection to LocalStack](<Screenshot 2026-07-29 101402.png>)

## Pre-Lab Verification Checklist

| Check | Status | Supporting evidence |
| --- | --- | --- |
| `kind --version` returns a version | Completed | Screenshot 1 |
| `kubectl version --client` returns a client version | Completed | Screenshot 1 |
| LocalStack health endpoint responds | Completed | Screenshots 2 and 7 |
| LocalStack lifecycle commands work | Completed | Screenshot 3 |
| Kind cluster `ccse` can be created | Completed | Screenshot 4 |
| `kubectl get nodes` reports a ready node | Completed | Screenshot 4 |
| Kind cluster `ccse` can be deleted | Completed | Screenshot 5 |
| AWS CLI dummy credentials and region are configured | Completed | Screenshot 5 |
| `oathtool --version` returns a version | Completed | Screenshot 6 |
| AWS CLI can call LocalStack STS | Completed | Screenshot 7 |

## Troubleshooting Notes from the Guide

| Symptom | Cause and recommended action |
| --- | --- |
| `Could not connect to the endpoint URL: http://localhost:4566/` | LocalStack is not running or port `4566` is not mapped. Run `docker ps`; if the container is absent, start it with `docker run -d --name localstack -p 4566:4566 localstack/localstack:3`. This was encountered and resolved in Step 7. |
| Port `4566` is already in use | An old LocalStack container may still own the port. Check Docker containers, then remove the old one with `docker rm -f localstack` before starting a new container. |
| `aws`, `kind`, `kubectl`, or `oathtool` is not recognised | The tool may not be installed or its installation directory may not be in `PATH`. Reinstall the tool if necessary and open a new terminal session so environment changes are loaded. |
| `kind create cluster` fails | Verify that Docker is running and has enough memory available. Since kind creates Kubernetes nodes as Docker containers, it cannot create a cluster without a working Docker engine. |
| Docker commands cannot connect to the daemon | Start Docker or, on Linux, ensure the current user is permitted to use Docker. A re-login may be required after changing Docker group membership. |
| One-time passwords are rejected in a later lab | Ensure that automatic system time synchronisation is enabled. TOTP codes rely on the client and server clocks being close to the same time. |

## Conclusion

The Lab 0 environment was prepared and verified on Kali Linux. The supplied evidence confirms installation and use of `kind`, `kubectl`, and `oathtool`; successful operation of LocalStack in Docker; creation and deletion of the `ccse` Kubernetes cluster; and successful AWS CLI communication with LocalStack using dummy credentials.

The environment is ready for the following IKB42603 lab exercises. In particular, LocalStack provides a safe local AWS-compatible endpoint, and the Kind cluster provides a local Kubernetes platform without requiring external cloud resources.
