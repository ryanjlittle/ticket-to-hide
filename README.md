# Ticket to Hide

__This repository contains an implementation of the TLS Oracle protocol in the work "Ticket to Hide: Practical, Private Proofs of Provenance for TLS".__

This code has been tested on MacOS Sequoia and Amazon Linux 2023.

## Repository Structure

This implementation is written in a combination of Python (for high-level networking and TLS operations) and C++ (for MPC computations). Our Python code contains a standalone TLS 1.3 implementation that supports Encrypted Client Hello and session resumption tickets. Our C++ code uses [EMP Toolkit](https://github.com/emp-toolkit) for MPC, and builds on top of the [Garble-then-Prove implementation](https://github.com/primus-labs/otls) built by [Primus Labs](https://primuslabs.xyz/).

This codebase is structured as follows:

```
ticket-to-hide                      
├── src                  
│   ├── python              : Python source code
│   │   ├── tls13           : Our Python TLS 1.3 implementation
│   │   └── tickettohide    : Our Python code implementing the high-level Ticket to Hide protocol
│   └── cpp                 : C++ source code 
│       ├── garblethenprove : The Primus Labs implementation of garble-then-prove. We do not modify this code
│       └── tickettohide    : Our C++ code implementing the MPC operations of Ticket to Hide
├── scripts                 : Bash scripts for building and deploying
└── benchmarks              : Bash scripts for reproducing experiments
```

## System Architecture

The top-level logic for the prover and verifier is written in Python and contained in the files `src/python/tickettohide/prover.py` and `src/python/tickettohide/verifier.py` respectively. To send messages between the prover and the verifier, the prover Python code listens on a port, and the verifier Python code connects to it. 

MPC is handled with a separate C++ program. To combine the C++ and Python into a single functionality, the prover and verifier Python code each spawn a subprocess to run the C++ code. The Python code then interacts with the C++ via stdin/stdout.

The core MPC logic for both the prover and verifier is contained in `src/cpp/tickettohide/tickettohide.cpp`. It accepts a command line argument to determine if it should run as the prover or the verifier. This program runs all the MPC operations of the protocol, using the garble-then-prove framework. It accepts secrets by reading them through stdin, and gives outputs by writing them to stdout. The prover and verifier versions of this program communicate with each other on a different port than the Python code. 

## Dependencies

Ticket to Hide requires the following dependencies. For each, we list an installation command for Mac and Amazon Linux:

- Python 3.13+
  - Mac: `brew install python@3.13`
  - Amazon Linux: `sudo yum install -y python3.13`
- C/C++ compiler and CMake
  - Mac: `xcode-select --install`
  - Amazon Linux: `sudo yum groupinstall "Development Tools"`
- CMake
  - Mac: `brew install cmake`
  - Amazon Linux: `sudo yum install -y cmake`
- OpenSSL Development Package
  - Mac: `brew install openssl`
  - Amazon Linux: `sudo yum install -y openssl-devel`
- EMP Toolkit (Primus Labs fork)
  - Installed by running the setup script below
- Various Python libraries
  - Installed by running the setup script below. For the full list of Python dependencies, see `src/python/requirements.txt`.
 
## Building Ticket to Hide

We provide a one-shot script for installation and compilation: `scripts/setup.sh`. To run, exectute
```
./scripts/setup.sh [debug|release]
```
Running with the debug flag will compile a debuggable C++ program and enable logging. We recommend running in release mode to replicate the experiments performed in the paper.

## Running Ticket to Hide

We provide Python scripts `src/python/tickettohide/run_prover.py` and `src/python/tickettohide/run_verifier.py` to run the full interaction. 



The prover script has two required arguments and several optional arguments. For a summary, the script can be run with the `-h` flag, which produces the following output:
```
usage: run_prover.py [-h] [-main_port [MAIN_PORT]] [-mpc_port [MPC_PORT]] [-rseed [RSEED]]
                     servers secrets [benchmark_file]

Runs the prover program

positional arguments:
  servers               File containing a list of server hostnames and ports
  secrets               File containing the index of the real server and all secret queries
  benchmark_file        CSV file for output results

options:
  -h, --help            show this help message and exit
  -main_port [MAIN_PORT]
                        Port for high-level communication with verifier
  -mpc_port [MPC_PORT]  Port for communicating with verifier for MPC computations
```

The two required arguments are `servers` file and a `secrets` file, which have specific format requirements. The servers file contains a list of _N_ servers, line-delineated, represented in the form <HOST>:<PORT>. We include an example in `src/python/servers_5.txt`. The secrets file contains the index of the prover's real server on the first line, followed by _N_ base64-encoded query messages for each server. We include an example in `src/python/servers_5.txt`, in which all queries are the same message.

By default, the main port is 8000, the MPC port is 8001, and no output results are logged. 

The verifier has one required argument, and its usage is as follows:

```
usage: run_verifier.py [-h] [-prover_host [PROVER_HOST]] [-main_port [MAIN_PORT]]
                       [-mpc_port [MPC_PORT]] [-rseed [RSEED]]
                       servers
positional arguments:
  servers               File containing a list of server hostnames and ports

options:
  -h, --help            show this help message and exit
  -prover_host [PROVER_HOST]
                        Prover hostname
  -main_port [MAIN_PORT]
                        Port for high-level communication with prover
  -mpc_port [MPC_PORT]  Port for communicating with prover for MPC computations
```

The `servers`, `main_port` and `mpc_port` arguments should be the same as the ones passed to the prover. By default, the main port is 8000, the MPC port is 8001.

#### Example protocol execution with test servers

To set up test servers, we provide the script `scripts/deploy_servers.sh`. This launches simultaneous servers on different ports running our TLS 1.3 implementation. It requires two arguments: the _number of servers_, and the _starting port_. All port numbers are assigned sequentially. To launch 5 servers on ports 9000-9004, run:

```
./scripts/deploy_servers.sh 5 9000
```

It can take a few seconds for the servers to start up. Wait until you see `Listening on port XXX` log messages before trying to connect.

In two separate terminals, navigate to the python directory and activate the virtual environment generated by the setup script as follows:
```
cd src/python
source venv/bin/activate
```

From one of these terminals, start the prover by running
```
python3 -m tickettohide.run_prover example/servers_5.txt example/secrets_5.txt 
```

Start the verifier on the other terminal by running
```
python3 -m tickettohide.run_verifier example/servers_5.txt
```

## Running Experiments

This section describes how to replicate the experiments performed in the paper. Experiments can either be run locally or over WAN. To run over WAN, three machines in different area networks are required.

The experimental logic is contained in the scripts `benchmarks/prover.sh` and `benchmarks/verifier.sh`. These scripts run complete Ticket-to-Hide protocol executions for a varying number of servers: 1, 10, 20, and up to 100 in increments of 10. In each execution, the prover uses a 128-byte query plaintext for all servers. The `prover.sh` script produces a csv file with output data, including timing measurements for each step of the protocol and communication measurements. Experiment scripts can be run with a chosen number of iterations per data point. 

To run, perform the following steps:

#### 1. Launch servers
Deploy 100 TLS servers on ports 9000-9099 by running
```
./scripts/deploy_servers.sh 100 9000
```

Wait until you see `Listening on port XXX` log messages before starting the prover/verfifier.

#### 2. Run prover 
Switch to the machine to run the prover, or open a new console if running locally. Set the `BENCHMARK_SERVER_IP` environment variable to the IP address of the machine running the servers. To run locally, use 0.0.0.0:
```
export BENCHMARK_SERVER_IP=0.0.0.0
```
To start the experiment with 10 iterations, run
```
./benchmarks/prover.sh 10
```

#### 3. Run verifier
Switch to the machine to run the prover, or open a new console if running locally. Now set the `BENCHMARK_SERVER_IP` environment variable to the same value used for the prover. Additionally, set the `BENCHMARK_PROVER_IP` environment variable to the IP address of the prover (or 0.0.0.0 for local testing):

```
export BENCHMARK_SERVER_IP=0.0.0.0
export BENCHMARK_PROVER_IP=0.0.0.0
```
Start the experiment with the same number of iterations as the prover:
```
./benchmarks/verifier.sh 10
```

Running the full experiment may take a few minutes. The results will be written to a file `benchmarks/results_XXX.csv` on the prover machine, where XXX is a UNIX timestamp.

__Note:__ The experiment scripts create a lot of temporary files. If you encounter a `OSError: [Errno 24] Too many open files` error during the experiment, increase the OS file descriptor limit by running `ulimit -n 1024`.
