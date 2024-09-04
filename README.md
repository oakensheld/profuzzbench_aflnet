# PinguFuzzBench - A Benchmark for Cryptographic Network Protocol Fuzzing

> [!NOTE] 
> **PINGU** is the abbreviation of cryptogra**P**hc network**ING** comm**U**nication protocols :)

PinguFuzzBench is a benchmark for the fuzzing of cryptographic network protocols. It is a specialized version of ProFuzzbench, which has removed some of the plaintext transmission network protocols from ProFuzzbench and added multiple encrypted network protocols (including TLS, SSH, QUIC, DTLS).

# Progress
***

| Protocol                            | **Pingu** | AFLNet | StateAFL | SGFuzz | FT-Net | tlspuffin | tlsfuzzer | tls-anvil |
| ----------------------------------- | --------- | ------ | -------- | ------ | ------ | --------- | --------- | --------- |
| [TLS/OpenSSL](subjects/TLS/OpenSSL) | ✅         | ✅      | ⌛        |        | ⌛      |           |           |           |
| [TLS/WolfSSL](subjects/TLS/WolfSSL) |           |        |          |        |        |           |           |           |
| SSH/OpenSSH                         |           |        |          |        |        |           |           |           |
| QUIC/OpenSSL                        |           |        |          |        |        |           |           |           |
| DTLS/OpenSSL                        |           |        |          |        |        |           |           |           |


# Folder structure
```
pingu-fuzzbench-folder
├── subjects: this folder contains all the different protocols included in this benchmark and
│   │         each protocol may have more than one implementations
│   └── TLS
│   │   └── OpenSSL
│   │       └── config.sh
│   │       └── ft-sink.yaml
│   │       └── ft-source.yaml
│   │       └── pingu-sink.yaml
│   │       └── pingu-source.yaml
│   │       └── README.md
│   │       └── ...
│   └── HTTP/3
│   └── SSH
│   └── ...
└── scripts: this folder contains all the scripts and configuration files to run experiments, collect & analyze results
│   └── build-env.sh: this file builds the docker image according to Dockerfile-env (**pingu-env**) that contains all the fuzzer binaries
│   └── Dockerfile-env
│   └── Dockerfile-dev: this file specifies the docker image that contains all the fuzzer source codes, dependencies and development environments
│   └── build.sh: this file builds the image for fuzzing runtime, based on the image **pingu-env**, according to Dockerfile. Each target should be built in a separate docker image using different fuzzers, like the image for TLS/OpenSSL instrumented and fuzzed by AFLNet, the image will be **pingu-tls-openssl-aflnet**
│   └── Dockerfile: this file builds the fuzzing runtime environment. The built image may be repeatedly launched to evaluate the fuzzer several times
│   └── run.sh: this file launches the fuzzing runtime container based on the image built by build.sh
│   └── evaluate.sh: this file will builds and launches the evaluation container, based on Dockerfile-eval, which includes jupyter, matplotlib and other stuff. The container is named with **pingu-eval**
│   └── Dockerfile-eval
│   └── utils.sh
│   └── shortcut.sh: some shortcuts for frequently used commands with some secrets
└── README.md: this file
```

# Fuzzers

ProFuzzBench provides automation scripts for fuzzing the targets with three fuzzers: [AFLnwe](https://github.com/aflnet/aflnwe) (a network-enabled version of AFL, which sends inputs over a TCP/IP sockets instead of files), [AFLNet](https://github.com/aflnet/aflnet) (a fuzzer tailored for stateful network servers), and [StateAFL](https://github.com/stateafl/stateafl) (another fuzzer for stateful network servers).

In the following tutorial, you can find instructions to run AFLnwe and AFLnet (the first two fuzzers supported by ProFuzzBench). For more information about StateAFL, please check out [README-StateAFL.md](README-StateAFL.md).


# Tutorial - Fuzzing TLS/OpenSSL server with [AFLNet](https://github.com/aflnet/aflnet)
Follow the steps below to run and collect experimental results for TLS/OpenSSL. The similar steps should be followed to run experiments on other subjects. Each subject program comes with a README.md file showing subject-specific commands to run experiments.

## Step-0. Prerequisites

- **Docker**: Make sure you have docker installed on your machine. If not, please refer to [Docker installation](https://docs.docker.com/get-docker/). The docker-engine that supports DOCKER_BUILDKIT=1 would be better, but it is not required.
- **Storage**: Also make sure you have enough storage space for the built images and the fuzzing results. Usually, the pingu-env image is around 3.3GB and the fuzzing runtime image is around 4.3GB, depending on the target program.


## Step-1. Build the base image

First change the working directory to the root directory of the repository.


```sh
./scripts/build-env.sh -- --build-arg HTTP_PROXY=http://127.0.0.1:9870 --build-arg HTTPS_PROXY=http://127.0.0.1:9870 --network=host --build-arg GITHUB_TOKEN=xxx 
```

The parameters specified after the **--** are the build arguments passed directly for the docker build command. You can specify sth like `--network=host --build-arg HTTP_PROXY=xxx`. Check the [Dockerfile-env](scripts/Dockerfile-env) to see the available build arguments.


## Step-2. Build the fuzzing runtime image

```sh
./scripts/build.sh -t TLS/OpenSSL -f ft -v 7b649c7 -- --network=host
```

The parameters specified after the **--** are the build arguments passed directly for the docker build command. You can specify sth like `--network=host --build-arg HTTP_PROXY=xxx`. Check the [Dockerfile](scripts/Dockerfile) to see the available build arguments.

Arguments:
- ***-t / --target*** : name of the target implementation (e.g., TLS/OpenSSL). The name should be referenced in the subjects directory.
- ***-f / --fuzzer*** : name of the fuzzer. Supports aflnet, stateafl, sgfuzz, ft, puffin, pingu.
- ***-v / --version*** : the version of the target implementation. Tag names and commit hashes are supported.

## Step-3. Fuzz

```sh
./scripts/run.sh -t TLS/OpenSSL -f ft -v 7b649c7 --times 1 --timeout 60 -o output
```

Required arguments:
- ***-t / --target*** : name of the target implementation (e.g., TLS/OpenSSL). The name should be referenced in the subjects directory.
- ***-f / --fuzzer*** : name of the fuzzer. Supports aflnet, stateafl, sgfuzz, ft, puffin, pingu.
- ***-v / --version*** : the version of the target implementation. Tag names and commit hashes are supported.
- ***--times*** : number of runs. The count of runs means the count of the docker containers.
- ***--timeout*** : timeout for each run, in seconds, like 86400 for 24 hours.
- ***-o / --output*** : output directory

Options:
- ***--cleanup*** : automatically delete the container after the fuzzing process.
- ***--detached*** : wait for the container to exit in the background.

## Step-4. Analyze the results

```sh
./scripts/evaluate.sh -t TLS/OpenSSL -f ft -v 7b649c7 -o output -c 2 -- --build-arg HTTP_PROXY=http://127.0.0.1:9870 --build-arg HTTPS_PROXY=http://127.0.0.1:9870 --network=host $script_args
```
The parameters specified after the **--** are the build arguments passed directly for the docker build command. You can specify sth like `--network=host --build-arg HTTP_PROXY=xxx`. Check the [Dockerfile-eval](scripts/Dockerfile-eval) to see the available build arguments.

Required arguments:
- ***-t / --target*** : name of the target implementation (e.g., TLS/OpenSSL). The name should be referenced in the subjects directory.
- ***-f / --fuzzer*** : name of the fuzzer. Supports aflnet, stateafl, sgfuzz, ft, puffin, pingu.
- ***-v / --version*** : the version of the target implementation. Tag names and commit hashes are supported.
- ***-o / --output*** : the directory where the results are stored.
- ***-c / --count*** : the number of runs to be evaluated upon.


# Utility scripts


# Parallel builds

To speed-up the build of Docker images, you can pass the option "-j" to `make`, using the `MAKE_OPT` environment variable and the `--build-arg` option of `docker build`. Example:

```
export MAKE_OPT="-j4"
docker build . -t lightftp --build-arg MAKE_OPT
```

# FAQs

## 1. Q1

## 2. Q2

## 3. Q3

# Citing PinguFuzzBench

# Citing ProFuzzBench

ProFuzzBench has been accepted for publication as a [Tool Demonstrations paper](https://dl.acm.org/doi/pdf/10.1145/3460319.3469077) at the 30th ACM SIGSOFT International Symposium on Software Testing and Analysis (ISSTA) 2021.

```
@inproceedings{profuzzbench,
  title={ProFuzzBench: A Benchmark for Stateful Protocol Fuzzing},
  author={Roberto Natella and Van-Thuan Pham},
  booktitle={Proceedings of the 30th ACM SIGSOFT International Symposium on Software Testing and Analysis},
  year={2021}
}
```