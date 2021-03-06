# pytorch-cheatsheet

A cheatsheet to cover the most commonly used aspects of PyTorch!

Pytorch Documentation: 
http://pytorch.org/docs/master/

Pytorch Forums: 
https://discuss.pytorch.org/

## Tensor Types and Operations

##### Commonly used types of tensors
`torch.IntTensor`

`torch.FloatTensor`

`torch.DoubleTensor`

`torch.LongTensor`

##### Common pytorch functions
```
torch.sigmoid(x)
torch.log(x)
torch.sum(x, dim=<dim>)
torch.div(x, y)
```
and many others: `neg(), reciprocal(), pow(), sin(), tanh(), sqrt(), sign()`

##### Convert numpy array to pytorch tensor
`b = torch.from_numpy(a)`

The above keeps the original dtype of the data (e.g. float64 becomes `torch.DoubleTensor`). To override this you can do the following (to cast it to a `torch.FloatTensor`):

`b = torch.FloatTensor(a)`

##### Moving data from CPU to GPU
`data = data.cuda()`

##### Pytorch Variable
The pytorch `torch.autograd.Variable` has a "data" tensor under it:
`data_variable.data`

##### Moving data from GPU to CPU
`data = data.cpu()`

##### Convert tensor to numpy array
`data_arr = data_tensor.numpy()`

##### Moving a Variable to CPU and converting to numpy array
`data = data.data.cpu().numpy()`

##### Viewing the size of a tensor
`data.size()`

##### Add a dimenstion to a torch tensor
`<tensor>.unsqueeze(axis)`

##### Reshaping tensors
The equivalent of numpy's `reshape()` in torch is `view()`

Examples:

```
a = torch.range(1, 16)
a = a.view(4, 4)        # reshapes from 1 x 16 to 4 x 4
```

`<tensor>.view(-1)` vectorizes a tensor.

##### Operations between Pytorch Variables and Numpy variables
The Numpy variables have to be first converted to `torch.Tensor` and then converted to pytorch `torch.autograd.Variable`. 
An example is shown below.

```
matrix_tensor = torch.Tensor(matrix_numpy)
# Use cuda() if everything will be calculated on GPU
matrix_pytorch_variable_cuda_from_numpy = torch.autograd.Variable(matrix_tensor, requires_grad=False).cuda()
loss = F.mse_loss(matrix_pytorch_variable_cuda, matrix_pytorch_variable_cuda_from_numpy)
```

##### Batch matrix operations
```
# Batch matrix multiply
torch.btorch.bmm(batch1, batch2, out=None)
```
##### Transpose a tensor
Transpose axis1 and axis2
`<tensor>.transpose(axis1, axis2)`

##### Outer product
Outer product between two vectors `vec1` and `vec2`
```
output = vec1.unsqueeze(2)*vec2.unsqueeze(1)
output = output.view(output.size(0),-1)
```

## Running on multiple GPUs

##### Multiple GPUs/CPUs for training

Instantiate the model first and then call DataParallel. todo: Add a way to specify the number of GPUs.

`model = Net()`

`model = torch.nn.DataParallel(model)`

For specifying the GPU devices: 
`model = torch.nn.DataParallel(model, device_ids=[0,1,2,3]).cuda()`

Pytorch 0.2.0 supports distributed data parallelism, i.e. training over multiple nodes (CPUs and GPUs)

##### Setting the GPUs

Usage of the `torch.cuda.set_device(gpu_idx)` is discouraged in favor of `device()`. In most cases it’s better to use the `CUDA_VISIBLE_DEVICES` environmental variable.



## Datasets and Data Loaders

##### Creating a dataset and enumerating over it
Inherit from `torch.utils.data.Dataset` and overload `__getitem__()` and `__len()__`

Example:

```
class FooDataset(torch.utils.data.Dataset):
  def __init__(self, root_dir):
    ...
  def __getitem__(self, idx):
    ...
    return {'data': batch_data, 'label': batch_label}
  def __len__(self):
    ...
    return <length of dataset>
```

To loop over a datasets batches:
```
foo_dataset = FooDataset(root_dir)
data_loader = torch.utils.data.DataLoader(foo_dataset, batch_size=<batch_size>, shuffle=True)

for batch_idx, batch in enumerate(data_loader):
  data = batch['data']
  label = batch['label']
  
  if args.cuda:
    data, label = data.cuda(), label.cuda()
    
  data = Variable(data)
  label = Variable(label)
```

## Setting Torch Random Seed
Use `torch.manual_seed(seed)` in addition to `np.random.seed(seed)` to make training deterministic. I believe in the future torch will use the numpy seed so they won't be separate anymore. 

## Convolutional Layers

##### Conv2d layer
`torch.nn.Conv2d(in_channels, out_channels, (kernel_w, kernel_h), stride=(x,y), padding=(x,y), bias=False, dilation=<d>)`


##### Transpose Conv2d layer ('Deconvolution layer')
`torch.nn.ConvTranspose2d(in_channels, out_channels, (kernel_w, kernel_h), stride=(x,y), padding=(x,y), output_padding=(x,y), bias=False, dilation=<d>)`


## Model Inference

##### Volatile at inference time
Don't forget to set the input to the graph/net to `volatile=True`. Even if you do `model.eval()`, if the input data is not set to volatile then memory will be used up to compute the gradients. `model.eval()` sets batchnorm and dropout to inference/test mode, insteasd of training mode which is the default when the model is instantiated. If at least one torch Variable is not volatile in the graph (including the input variable being fed into the network graph), it will cause gradients to be computed in the graph even if `model.eval()` was called. This will take up extra memory. 

Example:

`data = Variable(data, volatile=True)`

`output = model(data)`

## Saving/Loading Pytorch Models
Update: Pytorch now supports exporting models to other frameworks starting with Caffe2AI and MSCNTK. So now models can be deployed/served!

Pytorch v0.3.0: Model Exporter to ONNX (ship PyTorch to Caffe2, CoreML, CNTK, MXNet, Tensorflow)

Currently there is no supported way within Pytorch to serve/deploy models efficiently. 
For the sake of resuming training Pytorch allows saving and loading the models via two means - see http://pytorch.org/docs/master/notes/serialization.html. 
Beware that load/save pytorch models breaks down if the directory structure or class definitions change so when its time to deploy the model (and by this I mean running it purely from python on another machine for example) the model class has to be added to the python path in order for the class to be instantiated. It's actually very weird. If you save a model, change the directory structure (e.g. put the model in a subfolder) and try to load the model - it will not load. It will complain that it cannot find the class definition. The work around would be to add the class definition to your python path. This is written as a note on the Pytorch documentation page http://pytorch.org/docs/master/notes/serialization.html. I'll add an example on this later on.

## Loss Functions

A list of all the ready-made losses is here: http://pytorch.org/docs/master/nn.html#loss-functions

In Pytorch you can write any loss you want as long as you stick to using Pytorch `Variables` (without any `.data` unpacking or numpy conversions) and `torch` functions. The loss will not backprop (when using `loss.backward()`) if you use numpy data structures.

## Training an RNN with features from a CNN
Use `torch.stack(feature_seq, dim=1)` to stack all the features from the CNNs into a sequence. Then feed this into the RNN. Remember you can specify the batch size as the first dimension of the input tensor but you have to set the `batch_first=True` argument when instantiating the RNN (by default it is set to False).

Example:
```
 self.rnn1 = torch.nn.GRU(input_size=input_size, hidden_size=hidden_size, num_layers=2, batch_first=True)
```




