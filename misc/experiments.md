# README

This document describes the choke-point analysis, optimization recommendation, and end-to-end evaluation experiments.

## Environment and Datasets

Experiments 1 and 2 are conducted in Docker with the following environment.
- Docker image: `tugraph/tugraph-compile-centos8:1.3.1`
- Hardware: Alibaba Cloud `ecs.r6.8xlarge`, 32xIntel Xeon Platinum 8269CY vCPUs, 256GiB RAM

FinBench Datasets:
- [SF 0.1](oss://tugraph-web/tugraph/datasets/finbench/v0.2.0/sf0.1/sf0.1.tar.xz) with [md5 checksum](oss://tugraph-web/tugraph/datasets/finbench/v0.2.0/sf0.1/sf0.1.tar.xz.md5sum)
- [SF 1](oss://tugraph-web/tugraph/datasets/finbench/v0.2.0/sf1/sf1.tar.xz) with [md5 checksum](oss://tugraph-web/tugraph/datasets/finbench/v0.2.0/sf1/sf1.tar.xz.md5sum)
- [SF 3](oss://tugraph-web/tugraph/datasets/finbench/v0.2.0/sf3/sf3.tar.xz) with [md5 checksum](oss://tugraph-web/tugraph/datasets/finbench/v0.2.0/sf3/sf3.tar.xz.md5sum)
- [SF 10](oss://tugraph-web/tugraph/datasets/finbench/v0.2.0/sf10/sf10.tar.xz) with [md5 checksum](oss://tugraph-web/tugraph/datasets/finbench/v0.2.0/sf10/sf10.tar.xz.md5sum)
- [SF 30](oss://tugraph-web/tugraph/datasets/finbench/v0.2.0/sf30/sf30.tar.xz) with [md5 checksum](oss://tugraph-web/tugraph/datasets/finbench/v0.2.0/sf30/sf30.tar.xz.md5sum)
- [SF 100](oss://tugraph-web/tugraph/datasets/finbench/v0.2.0/sf100/sf100.tar.xz) with [md5 checksum](oss://tugraph-web/tugraph/datasets/finbench/v0.2.0/sf100/sf100.tar.xz.md5sum)

## Experiment 1: Temporal Locality in Storage

The temporal locality experiment is executed on TuGraph-DB with FinBench SF100 dataset. The following steps show how to reproduce the temporal locality experiment. Execute following steps under current doc's directory `ldbc_finbench_transaction_impls/misc`.

### 1. Download Experiment Resources

Download temporal locality experiment package into current directory and download the FinBench sf100 dataset into experiment package directory.

```shell
$ wget https://tugraph-web.oss-cn-beijing.aliyuncs.com/tugraph/datasets/finbench/v0.2.0/experiments/temporal_locality.zip
$ unzip temporal_locality.zip
$ cd ./temporal_locality
$ wget https://tugraph-web.oss-cn-beijing.aliyuncs.com/tugraph/datasets/finbench/v0.2.0/sf100/sf100.tar.xz
$ tar -xf ./sf100.tar.xz
$ cd ..
```

### 2. Update Source Code in Query Implementation

The experiment uses procedure to execute `ComplexRead1` instead of Cypher, which requires a modification in the `ComplexRead1Handler` in `ldbc_finbench_transaction_impls/tugraph/src/main/java/org/ldbcouncil/finbench/impls/tugraph/TuGraphTransactionDb.java:100` with the following code.

```java
public static class ComplexRead1Handler implements OperationHandler<ComplexRead1, TuGraphDbConnectionState> {
        @Override
        public void executeOperation(ComplexRead1 cr1, TuGraphDbConnectionState dbConnectionState,
                ResultReporter resultReporter) throws DbException {
            try {
                TuGraphDbRpcClient client = dbConnectionState.popClient();
                String cypher = "CALL plugin.cpp.tcr1({ id:%d, startTime: %d, endTime: %d, truncationLimit: %d});";
                long startTime = cr1.getStartTime().getTime();
                long endTime = cr1.getEndTime().getTime();
                int truncationLimit = cr1.getTruncationLimit();
                cypher = String.format(
                        cypher,
                        cr1.getId(), startTime, endTime, truncationLimit);
                String graph = "default";
                String res = client.callCypher(cypher, graph, 0);
                ArrayList<ComplexRead1Result> results = new ArrayList<>();
                JSONArray array = JSONObject.parseArray(res);
                for (int i = 0; i < array.size() - 1; i++) {
                    JSONObject ob = array.getJSONObject(i);
                    ComplexRead1Result result = new ComplexRead1Result(
                            ob.getLongValue("otherId"),
                            ob.getIntValue("accountDistance"),
                            ob.getLongValue("mediumId"),
                            ob.getString("mediumType"));
                    results.add(result);
                }
                resultReporter.report(results.size(), results, cr1);
                dbConnectionState.pushClient(client);

                JSONObject ob = array.getJSONObject(array.size() - 1);
                System.out.printf("iterationTime: %f, executionTime: %f%n", ob.getDoubleValue("iterationTime"), ob.getDoubleValue("executionTime"));
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

### 3. Compile Benchmark Implementation

Compile the implementation.

```shell
$ cd ..
$ mvn clean package
$ cd ./misc
```

### 4. Install TuGraph-DB in Docker Container

To run temporal locality experiment in docker `tugraph/tugraph-compile-centos8:1.3.1`, install `tugraph-4.0.0-1.el8.x86_64.rpm` first.

```shell
$ docker pull tugraph/tugraph-compile-centos8:1.3.1
$ docker run -it -v ./temporal_locality:/root/temporal_locality -p 7071:7071 -p 9091:9091 --name=finbench_temporal_loaclity tugraph/tugraph-compile-centos8:1.3.1 /bin/bash
$ cd /root/temporal_locality/
$ rpm -ivh tugraph-4.0.0-1.el8.x86_64.rpm --nodeps
```

Use the following command to check if it is successfully installed, whose output is like this. 

```shell
$ lgraph_server --version
TuGraph v4.0.0, compiled from "finbench" branch, commit "335335f" (web commit "8253763").
  CPP compiler version: "GNU" "8.4.0".
  Python version : "3.6.9".
```

### 5. Preprocess SF100 Dataset

Before importing SF100 dataset，we need to preprocess the snapshot of SF100 dataset in terms of the data format. This can be done outside of docker container.

```shell
$ cd ./temporal_locality/sf100
$ mv ./snapshot ./snapshot.bak
$ mkdir ./snapshot
$ cd ..
$ python convert_data.py ./sf100/snapshot.bak ./sf100/snapshot
$ cd ..
```


### 6. Run Baseline Version

Import SF100 dataset with temporal sorting feature disabled. Then start TuGraph-DB server.

```shell
$ docker exec -it finbench_temporal_loaclity /bin/bash
$ cd /root/temporal_locality/sf100/snapshot
$ lgraph_import -c /root/temporal_locality/baseline/import.conf --overwrite 1 --dir /root/lgraph_db_sf100 --delimiter "|" --v3 0
$ cd /root/temporal_locality/
$ lgraph_server -c /root/temporal_locality/lgraph_sf100.json -d start
$ exit # exit from the container
```

Load `ComplexRead1` procedure (baseline version) for baseline experiment. Then link sf100 dataset to `./data` directory, and run benchmark

```shell
$ cp ./temporal_locality/load_procedure_temporal.sh ../tugraph/scripts
$ chmod u+x ../tugraph/scripts/load_procedure_temporal.sh
$ cp ./temporal_locality/baseline/tcr1.cpp ../tugraph/procedures/cpp
$ ../tugraph/scripts/load_procedure_temporal.sh
$ cd ../tugraph/
$ cd ./data
$ ln -s ../../misc/temporal_locality/sf100 ./sf100
$ cd ..
$ bash run.sh ../misc/temporal_locality/benchmark.properties > temporal_locality.log
$ cd ../misc
```

The execution and iteration time in `ComplexRead1` is recorded in `temporal_locality.log`.
Collect and analyze the elapsed time with `./temporal_locality/log_data_collector.py`

```shell
$ cd ./temporal_locality
$ python log_data_collector.py ../../tugraph/temporal_locality.log
$ cd ..
```

Stop TuGraph-DB server.

```shell
$ docker exec -it finbench_temporal_loaclity /bin/bash
$ cd /root/temporal_locality/
$ lgraph_server -d stop
$ exit
```

### 7. Run Optimized Version

Import SF100 dataset with temporal sorting feature enabled. Then start TuGraph-DB server.

```shell
$ docker exec -it finbench_temporal_loaclity /bin/bash
$ cd /root/temporal_locality/sf100/snapshot
$ lgraph_import -c /root/temporal_locality/optimized/import.conf --overwrite 1 --dir /root/lgraph_db_sf100 --delimiter "|" --v3 0
$ cd /root/temporal_locality/
$ lgraph_server -c /root/temporal_locality/lgraph_sf100.json -d start
$ exit # exit from the container
```

Load `ComplexRead1` procedure (optimized version) for optimized experiment. Then link sf100 dataset to `./data` directory, then run temporal locality benchmark.

```shell
$ cp ./temporal_locality/load_procedure_temporal.sh ../tugraph/scripts
$ chmod u+x ../tugraph/scripts/load_procedure_temporal.sh
$ cp ./temporal_locality/optimized/tcr1.cpp ../tugraph/procedures/cpp
$ ../tugraph/scripts/load_procedure_temporal.sh
$ cd ../tugraph/
$ cd ./data
$ ln -s ../../misc/temporal_locality/sf100 ./sf100
$ cd ..
$ bash run.sh ../misc/temporal_locality/benchmark.properties > temporal_locality.log
$ cd ../misc
```

The execution and iteration time in `ComplexRead1` is recorded in `temporal_locality.log`.
Collect and analyze the information with `./temporal_locality/log_data_collector.py`

```shell
$ cd ./temporal_locality
$ python log_data_collector.py ../../tugraph/temporal_locality.log
$ cd ..
```

Stop TuGraph-DB server.

```shell
$ docker exec -it finbench_temporal_loaclity /bin/bash
$ cd /root/temporal_locality/
$ lgraph_server -d stop
$ exit
```

## Experiment 2: Recursive Path Filtering

### 1. Preparation

Download SF100 dataset and parameter. Move them to the dataset directory, e.g., the directory `/data/dataset`.

Modify docker-compose.yml to attach local file to volume `/root/datasets/` in container, for example, 
```
volumes: 
    - /data/dataset:/root/datasets/
    - /data/scripts:/root/scripts/
```

Start docker containers with `docker-compose up -d`. After that, two containers start.
- [directory name]_tugraph_1
- [directory name]_finbench_1


### 2. Benchmark Baseline

- Attach to the `xxx_tugraph_1` container. Then install the package with `rpm -i base.rpm`.
- In the container `xxx_tugraph_1`, Enter the `/root/scripts` directory. Import data with `bash sf100_import.sh` and start the server with `bash sf100_start.sh`
- Switch to another container `xxx_finbench_1`. Enter the `/root/scripts` directory. Install C++ plugins with `bash load_procedure.sh`
- In the container `xxx_finbench_1`, copy the file `benchmark_sf100.properties` in scripts to the directory `/root/ldbc/ldbc_finbench_transaction_impls/tugraph/benchmark.properties`.
- In the container `xxx_finbench_1`,  enter the `/root/scripts` directory and run the benchmark with `bash sf100_benchmark.sh`.

### 3. Benchmark Optimization

- Attach to the `xxx_tugraph_1` container. Uninstall the baseline package with `rpm -e tugraph-4.0.0-1.x86_64` and then install the package with `rpm -i opt.rpm`.
- In the container `xxx_tugraph_1`, enter the `/root/scripts` directory. Delete the old data directory with `rm -r /root/lgraph_db_sf100` and `rm -r /root/lgraph_log_sf100`.
- Import data with `bash sf100_import.sh` and start the server with `bash sf100_start.sh`
- Switch to another container `xxx_finbench_1`. Enter the `/root/scripts` directory. Install C++ plugins with `bash load_procedure.sh`
- In the container `xxx_finbench_1`, copy the file `benchmark_sf100.properties` in scripts to the directory
  `/root/ldbc/ldbc_finbench_transaction_impls/tugraph/benchmark.properties`.
- Comment the line in the file `sf100_benchmark.properties`.
```
# cd data && ln -s /root/datasets/sf100 ./sf100 && cd ..
```
- In the container `xxx_finbench_1`, enter the `/root/scripts` directory and run the benchmark with `bash sf100_benchmark.sh`.

## Experiment 3: Optimization Recommendations

### Galaxybase

#### 1. Configure the Experiment

Deploy Galaxybase, start the graph service, and install the FinBench procedures. Import the SF300 dataset and prepare its read and incremental parameters. Use the optimization-enabled driver and the slowed `SimpleRead6` implementation for this experiment.

Copy the supplied benchmark configuration to `optimization-sf300.properties`. Keep the connection and workload settings, select the SF300 graph and parameter paths, and update the following properties:

```properties
mode=OPTIMIZATION_RECOMMENDATION
time_compression_ratio=0.0075
results_dir=results/optimization-sf300
results_log=true

# Number of top-ranked queries to evaluate
optimization.top_n=1
# Latency reduction ratios for synthetic scenarios
optimization.reduction_steps=0.05,0.10,0.20
# Profiling repetitions and workload repetitions per scenario
optimization.profile_repetitions=3
optimization.repetitions=1
# Resource sampling interval in milliseconds
optimization.sample_interval_millis=1
```

#### 2. Run the Experiment

Run the configuration from the driver directory. The driver executes the baseline, ranks queries, and evaluates the configured synthetic acceleration scenarios.

```shell
cd /galaxybase/driver
./run_driver optimization-sf300.properties
```

Then restore the original `SimpleRead6` implementation. Copy the configuration to `benchmark-sf300-original.properties`, remove the `optimization.*` entries, and set:

```properties
mode=EXECUTE_BENCHMARK
results_dir=results/sf300-original
```

Keep the same TCR, thread count, warmup count, operation count, and workload parameters, then run:

```shell
./run_driver benchmark-sf300-original.properties
```

### GPStore

To be added.

## Experiment 4: End to End Evaluation

### Galaxybase

- [Download](https://drive.google.com/file/d/1KpjjhwnYN2NEaJ0_NOAh0gikpF24ecov/view?usp=sharing)

- [Galaxybase Official Website](https://createlink.com)

#### 1. Deploy Galaxybase

1. Extract the Galaxybase installation package.

```shell
tar -xzf galaxybase-*.tar.gz
```

2. Enter the extracted directory and edit the deployment configuration file at `./galaxybase-20260923141832/run/conf/galaxybase.yaml`:

```shell
cd galaxybase-*/

vim ./run/conf/galaxybase.yaml

# Set the data path
data: /galaxybase
```

3. Start the Galaxybase cluster management service.

```
cd galaxybase-20260923141832/
sudo ./galaxybase start admin

# Open the HTTP ports
sudo ufw allow 51314/tcp
sudo ufw allow 18088/tcp
sudo ufw allow 8888/tcp
```

4. Start the graph service.

```
sudo ./galaxybase start graph
```

5. Copy the LDBC Finbench stored procedures to `/galaxybase/graph/data/procedures`:

```shell
mkdir -p /galaxybase/graph/data/procedures
cp ./procedures/* /galaxybase/graph/data/procedures
```

6. Open the cluster management console at `http://hostname:51314`. Use `admin` as both the username and password. Go to **Cluster** → **Add Node**, then configure and start the graph service nodes for your deployment.

7. License verification is required during the first startup. Submit the request code to the Galaxybase team to obtain a license code.

8. Enter the license code provided by the Galaxybase team, accept the user agreement, and start license verification. After verification succeeds, the node status changes to `Starting...`.

9. When the node status changes to `Running`, the graph service has started successfully.

#### 2. Download the Datasets

Download the pre-generated LDBC Finbench SF3000 datasets from the official LDBC dataset page. 

After downloading, extract the dataset archives into `/galaxybase/graph/data`.

#### 3. Import a Dataset

1. Run `./load_graph.sh` to import the specified dataset. The `/galaxybase/graph/data` host path differs from the path inside the Docker container, so use the container-relative path `./data`. For example, if the dataset path outside the container is `/galaxybase/graph/data/sf3000/`, pass `./data/sf3000/snapshot` to the import script.

```
./load_graph.sh ./data/sf3000/snapshot
```

2. After the import starts, monitor its progress in the graph service log at `/galaxybase/graph/logs/sys/sys.log`.

```shell
tail -f /galaxybase/graph/logs/sys/sys.log
```

#### 4. Run the Benchmark

After the dataset import finishes on the graph service, run `run_driver` to start the benchmark.

```shell
cd /galaxybase/driver
./run_driver benchmark-sf3000.properties
```

#### 5. Run Validation

The `validate.properties` validation configuration file is provided for SF10.

After the dataset import finishes on the graph service, run the following command to start validation:

```shell
./run_driver validate.properties
```

### AtlasGraph

To be added.
