### Description
A small tool to dump Caffe's \*.caffemodel and \*.binaryproto files to JSON for inspection (\*.prototxt files is not needed). By default [caffe.proto](https://raw.githubusercontent.com/BVLC/caffe/master/src/caffe/proto/caffe.proto) from the official Caffe repository is used. Python and protobuf with Python bindings are dependencies.



### Modification

​	由于现多用python3.x，对原代码做少许修改为python3.x的版本。

- long -> int

  python2.x中同时存在long和int类型，python3.x中均未int

- unicode -> str

  python3.x中str即为python2.x中unicode

- import caffe_pb2 -> import caffe.proto.caffe_pb2 as caffe_pb2

  库版本变化

  

### Usage 

```shell
# to dump model structure without weights (unless `--data` switch is used) to JSON, using default caffe.proto:
./caffemodel2json.py model.caffemodel > dump.json

# to dump model structure without weights (unless `--data` switch is used) to JSON:
./caffemodel2json.py model.caffemodel CAFFE_ROOT/src/caffe/proto/caffe.proto > dump.json

# to dump a binaryproto:
wget http://dl.caffe.berkeleyvision.org/caffe_ilsvrc12.tar.gz
mkdir -p imagenet_mean && tar -xf caffe_ilsvrc12.tar.gz -C imagenet_mean
./caffemodel2json.py imagenet_mean/imagenet_mean.binaryproto --data > imagenet_mean.json
```

### 

### License

MIT
