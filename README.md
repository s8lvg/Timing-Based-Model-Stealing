# Timing based neural network parameter inferrence
A simple PoC project that shows that it is possible to infer information about neural networks timing behaviour. The experiments show that an advesary can trigger edge cases whith long timings and infer the network depth by only performing black box queries to the netowork.

## Before running
- These instructions only work on Linux for all other OS your on your own
- For best performance run on bare metal, else use a linux VM


To achieve precise timing measurements on your system close all other applications that are running (especially browsers). To further denoise your system disable turbo boose/core by running
```shell
# On INTEL
echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo
# On AMD
echo "0" | sudo tee /sys/devices/system/cpu/cpufreq/boost
```
Then set your cpu freqency to performance mode by executing the following line in your shell 
```shell
sudo cpupower frequency-set -g performance
```
## Iris Net

A simple implementation of a neural network that does classification on the iris dataset. The resulting network gets dumped in onnx format.

## Timing Blackbox

A rust network evaluator that takes the onnx implemtation of iris_net (or with small modifications any other neural network). Run it with `cargo +nightly run <path-to-onnx-model>` while in the `timing_blackbox` directory. It consists of the following parts.

### Webserver

Handles incoming request to the neural network all communication is done in json format and works like this

- Client sends a post request of the form

```json
{"input_values" : [<float>,<float>,<float>,<float>]}
```

to the prediction API at `http://127.0.0.1/predict`

- Sever processes request and sends a response in the form

```json
{"confidence_vector":[<float>,<float>,<float>],"prediction_time":<float>}
```

back to the client

### Prediction API

The prediction api executed the loaded network and returns a confidence vector. In addition execution time of the network is measured in ns

```rust
// Start timestamp
let start = rdtscp();
// Model gets executed
let result = model_extract.run(tvec!(input_tensor)).unwrap();
// End timestamp
let end = rdtscp();
```

This prediction time then gets send to the client.

### Loading API

The loading api at `http://127.0.0.1/loadmodel` allows the client to load a different model into the blackbox. This is useful to compare execution times of different models with each other. To request a new model the client sends a post request containing

```json
{"path":<path-to-model>}
```

**TODO**: Include model input dimensions such that different models can easily be loaded.

## Genetic Advesary

A sample implementation that tries to maximise the time it takes for the network to execute. It works by using a genetic algorithm in the following way

- Sends request to the Prediction api
- Uses prediction_time value to compute a fitness function
- Mutate seeds to optimize fitness using genetic algorithm

## Layer detection Advesary

An adveary that uses ridge regression to infer the network depth from timing measurements.

- Profiles the network execution time for different depths to get an image of hardware execution speed. For this run set `PROFILE=TRUE`.
- Trains a ridge regression classifier on the results
- Generates a on the fly test set by quering and measuiring the time for randomly deeep neural networks of the same architecture.
- Predicts the test set and plots a confusion matrix

Results: ~ 90% accuracy, im sure it is possible to get close to 100% by tweaking hyperparameters.

# Troubleshoot

The timing blackbox uses the assembly instruction `rdscp` which may not be availiable on all systems. In case this leads to an error replace all calls to `rdscp()` with calls to `rdsc()`.

In order to debug the timing blackbox start it with the following arguments

```
RUST_LOG=trace cargo +nightly run
```

You can manually query the prediction server from the command line for debug purposes using

```
wget -O- --post-data='{"input_values":[0.0, 0.0, 0.0, 0.0]}'   --header='Content-Type:application/json'  'http://localhost:8080/predict' > result
```

Get the results using

```
cat result
```
