# TFaaS: from installation to operation

TFaaS will serve predictions for trained TF models via HTTPs. 

## What this doc is, what this doc is not

This is not intended to be a complete, fully tested and working documentation for everything about TFaaS. This is a set of notes for anyone who is willing to look closer to how working on TFaaS might look like. These instructions should help you to get a copy of the project up and running on your local machine, just for development and testing purposes. You might need to contact vkuznet or bonacor to get more information.

## TFaaS installation

To operate TFaaS, you need first to install it. The code can be found [here](https://github.com/vkuznet/TFaaS).

## From Keras to Tensorflow

This part assumes you have created and trained a model with Keras beforhand. This must be saved on local disk as a `.h5` file e.g. via `model.save('file_name.h5')`. The model must be converted to Tensorflow (TF) using the `keras_to_tensorflow` package, which can be found [here](https://github.com/vkuznet/keras_to_tensorflow.git). This package loads a trained keras model, freezes the nodes (converts all TF variables to TF constants), and saves the inference graph and weights into protobuf files. There are two different formats that a ProtoBuf can be saved in: text format (`.pbtxt` extension) files are in a human-readable form, which makes it nice for debugging and editing, but can get large when there is numerical data like weights stored in it; binary format (`.pb` extension) files are a lot smaller than their text equivalents, even though they are not as readable. In our case, reference commands that are needed are e.g.:
```
./keras_to_tensorflow.py -input_model_file [your_saved_model.h5] \
                         -output_model_file [output.pb]
./keras_to_tensorflow.py -input_model_file [your_saved_model.h5] \
                         -output_model_file [output.pbtxt]
```
This file (any format) can then be used to deploy the trained model for inference. During freezing, other nodes of the network, which do not contribute the tensor that would contain the output predictions, are pruned (this results in a smaller, optimized network).

## Go

You need to install the [Go (golang)](https://golang.org/) language on your system. To do so, the Go language package for your favorite distribution must be downloaded and installed, the `GOROOT` environment variable must be set to point to the installed go distribution, and the `GOPATH` environment variable must be set to point to your local area where you Go packages will be stored. 

Follow these simple instructions:
- download Go language for your favorite distribution, grab appropriate 
tar-ball from [here](https://golang.org/dl/)
- setup `GOROOT` to point to isntalled go distribution, e.g.
`export GOROOT=/usr/local/go`, and setup `GOPATH` environment
to point to your local area where you go packages will be stored, e.g.
`export GOPATH=/path/gopath`
(please verify that `/path/gopath` directory exists and creates it if necessary)
- obtain GO tensorflow library and install it on your system, see
this [instructions](https://www.tensorflow.org/versions/master/install/install_go)
Once installed you'll need to get go tensorflow code. It will be compiled
against library you just downloaded and installed, please verify that
you'll setup proper `LD_LIBRARY_PATH` (on Linux) or `DYLD_LIBRARY_PATH` (on OSX).
To get go tensorflow librarys you just do
```
go get github.com/tensorflow/tensorflow/tensorflow/go
go get github.com/tensorflow/tensorflow/tensorflow/go/op
```
- download necessary dependencies for `tfaas`:
```
go get github.com/vkuznet/x509proxy
go get github.com/golang/protobuf
go get github.com/sirupsen/logrus
```
- build `tfaas` Go server by running `make` and you'll get `tfaas` executable
  which is ready to server your models.
  
## Go TF library

You need to install the Go Tensorflow library on your system. A good instruction set can be found [here](https://www.tensorflow.org/versions/master/install/install_go). Download and extract the TensorFlow C library into e.g. `/usr/local/lib` by invoking the following shell commands:
```
TF_TYPE="cpu" # Change to "gpu" for GPU support
 TARGET_DIRECTORY='/usr/local'
 curl -L \
   "https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-\
   ${TF_TYPE}-$(go env GOOS)-x86_64-1.6.0-rc1.tar.gz" |
 sudo tar -C $TARGET_DIRECTORY -xz
```
Once installed youneed to get Go TF code. It will be compiled against the library you just downloaded and installed: a proper set-up of ``LIBRARY_PATH`` variables should be checked. Now that the TensorFlow C library is installed, invoke `go get` as follows to download the appropriate packages and their dependencies:
```
go get github.com/tensorflow/tensorflow/tensorflow/go
go get github.com/tensorflow/tensorflow/tensorflow/go/op
```
Download necessary dependencies for TFaaS:
```
go get github.com/vkuznet/x509proxy
go get github.com/golang/protobuf
go get github.com/sirupsen/logrus
```
Invoke `go test` as follows to validate the TensorFlow for Go installation:
```
go test github.com/tensorflow/tensorflow/tensorflow/go
```
Another way to check that Go is properly working (without which the TFaaS Go server will not work) is to try a simple Hello World in Go. A simple Go code like this:
```
package main

import (
    tf "github.com/tensorflow/tensorflow/tensorflow/go"
    "github.com/tensorflow/tensorflow/tensorflow/go/op"
    "fmt"
)

func main() {
    // Construct a graph with an operation that produces a string constant.
    s := op.NewScope()
    c := op.Const(s, "Hello from TensorFlow version " + tf.Version())
    graph, err := s.Finalize()
    if err != nil {
        panic(err)
    }

    // Execute the graph in a session.
    sess, err := tf.NewSession(graph, nil)
    if err != nil {
        panic(err)
    }
    output, err := sess.Run(nil, []tf.Output{c}, nil)
    if err != nil {
        panic(err)
    }
    fmt.Println(output[0].Value())
}
```
can be run via the command
```
go run hello_tf.go
```
and should print out an hello message with the installed TF version number. Once all this is done, you should make sure that produced executable is linked against TF library, e.g. via `ldd` command.

## Get TFaaS executables

Now you can build TFaaS Go server by running `make` and you will get the TFaaS executable which is ready to act as a server for your models. 

## Prepare to read models 

The `.pb` files (i.e. TF model files in protobuf data-format) should now be moved to the same node where TFaaS is installed, to be ready for the next step. Note that each TF model use input entry-point to feed the data (vector or matrix) into TF model. This entry point has a name either assigned by TF code or via explicit assignment. One may inspect the TF model in text format to find out which layer names where used and hence extract the input layer name and the output layer name.

## https set-up

In order to serve TF models with the TFaaS Go server, one needs to set-up server certificates to start HTTPs server. Best is to generate self-signed host certificates. An example how to generate self-signed certificates is:
```
openssl req -new -newkey rsa:2048 -nodes \\
        -keyout server.key -out server.csr
```
Varius arguments will needed to be asked, but they are self-explanatory.

## Prepare a prediction labels file

Attention must be put also to prediction labels. For models where there are multiple labels we need to create prediction labels file. It is a simple text file which lists its label on every line.

## Run the server

Once all this is done, all pieces are in place to run a TFaaS server. This can be done via this command:
```
openssl req -new -newkey rsa:2048 -nodes -keyout server.key -out server.csr
```
At this stage, after proper set-up of the user credential, the server can be run:
```
./tfaas -dir [dir_with_models] -serverCert server.csr -serverKey server.key -modelLabels [label_file] -inputNode [input_layer_name] -outputNode [input_layer_name] -modelName [model.pb]
```
One can see the server running:
```
INFO[0000] Starting HTTP server    Addr=":8083"
```
Once the server runs, one can prepare in any language a JSON with the records than one wants to send to the server, and send it like this:
```
# host settings
headers="Content-type: application/json"
host=https://localhost:8083
curl -k --key ~/.globus/userkey.pem --cert ~/.globus/usercert.pem -H "$headers" -d @/tmp/[my_json_file] $host/json
```
and the server will reply with predictions according to the trained model. In a realistic use-case (Luca Giommi's thesis, no reference yet as the dissertation is still due), a single one-shot call from the client to a running server looks e.g. like this:
```
$ curl -s -L -k --key ~/.globus/userkey.pem --cert ~/.globus/usercert.pem \\
-H "Content-type: application/json" -d '{"keys": ["nJets", "nLeptons", \\
"jetEta_0", "jetEta_1", "jetEta_2", "jetEta_3", "jetEta_4", "jetMass_0", \\
"jetMass_1", "jetMass_2", "jetMass_3", "jetMass_4", "jetMassSoftDrop_0", \\
"jetMassSoftDrop_1", "jetMassSoftDrop_2", "jetMassSoftDrop_3", \\
"jetMassSoftDrop_4", "jetPhi_0", "jetPhi_1", "jetPhi_2", "jetPhi_3", \\
"jetPhi_4", "jetPt_0", "jetPt_1", "jetPt_2", "jetPt_3", "jetPt_4", \\
"jetTau1_0", "jetTau1_1", "jetTau1_2", "jetTau1_3", "jetTau1_4", \\
"jetTau2_0", "jetTau2_1", "jetTau2_2", "jetTau2_3", "jetTau2_4", \\
"jetTau3_0", "jetTau3_1", "jetTau3_2", "jetTau3_3", "jetTau3_4"], "values": [2.0, 0.0, 0.9228423833849999, -1.1428750753399999, 0.0, 0.0, 0.0, 155.239425659, 142.709609985, 0.0, 0.0, 0.0, 83.5365982056, 120.549507141, 0.0, 0.0, 0.0, 1.9305502176299998, -1.17742347717, 0.0, 0.0, 0.0, 481.419799805, 449.04394531199995, 0.0, 0.0, 0.0, 0.296700358391, 0.286615312099, 0.0, 0.0, 0.0, 0.164555206895, 0.19625715911400002, 0.0, 0.0, 0.0, 0.117722302675, 0.155229091644, 0.0, 0.0, 0.0]}' http://localhost:8083/json
```
and the outcome on screen might look like
```
[0.65876985]
```
As a response, a floating point number is hence returned: in this specific use-case (see Luca's thesis) this means that that specific event is predicted by the trained model to be a Signal event ($>0.5$) and not a Background event. Tests done with large set of events show that the results obtainable locally with a given model are the same ones that are returned by the server via this kind of calls. Of course, for plenty of events this is not done command-line, but all event entries are passed via JSON format to the server in a unique, large client query, returning a vector of predictions. The predictions out of TFaaS will be consistent with those from the same model run locally.

## parse a JSON

(FIXME) to do: this part needs notes.
```
python parse_luca.py --fin=dataset_pro.csv
```


