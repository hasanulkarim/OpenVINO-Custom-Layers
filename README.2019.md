# OpenVINO Custom Layer Tutorial

## Before You Start
It's assumed that you've installed `OpenVINO 2019.r1`, including the Model Optimizer, in the default /opt/intel directory,.  If you're using an earlier version, refer to this [document](./README.md).  If you've installed to another directory, you may have to make changes to some of the commands.

---

Sample code, specifically the `classification_sample`, is located here:<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`~/inference_engine_samples/intel64/Release`<br>

The Model Optimizer is abbreviated `MO` for the remainder of this document.

There are 2 directories with C++ and Python source code for the cosh layer. 

### Custom Layers
Custom layers are NN (Neural Network) layers that are not explictly supported by a given framework.  This tutorial demonstrates how to run inference on topologies featuring custom layers. This way you can plug in your own implementation for existing or completely new layers.

The list of known layers is different for any particular framework. To see the layers supported by OpenVINO, refer to the [OpenVINO MO Documentation](https://software.intel.com/en-us/articles/OpenVINO-ModelOptimizer#intermediate-representation-notation-catalog).  If your topology contains any layers that are not in the list of known layers, the Model Optimizer considers them to be custom.

Model Optimizer searches for each layer of the input model in the list of known layers before building the model's internal representation, optimizing the model and producing the Intermediate Representation.

### Example custom layer- Hyperbolic Cosine (cosh) function 
We showcase custom layer implementation using a simple function, hyperbolic cosine (cosh). Mathematically, it is represented as:

![](https://latex.codecogs.com/gif.latex?cosh%28x%29%3D%5Cfrac%7Be%5E%7Bx%7D&plus;e%5E%7B-x%7D%7D%7B2%7D)


---

1. Prep and git clone this repository:<br>
    ```
    cd ~
    ```
    ```
    mkdir cl_tutorial
    ```
    ```
    cd cl_tutorial
    ```
    ```
    git clone https://github.com/david-drew/OpenVINO-Custom-Layers.git
    ```

2. Setup your environment for OpenVINO:<br>
    ```
    source /opt/intel/openvino/bin/setupvars.sh 
    ```

3. Install prerequisites (code generator for running Python snippets):
    `sudo pip3 install cogapp`

4. Run the MO extension generator and answer questions as appropriate: 
    * We're using `~/cl_cosh` as the target extension path:<br><br>
    * This will create templates that will be partially replaced by Python and C++ code for executing the layer.
    ```
    python3 /opt/intel/openvino/deployment_tools/extension_generator/extgen.py new mo-op ie-cpu-ext output_dir=~/cl_cosh
    ```
    * Answer questions as follows:
    ```
    1.  Is your layer Pythonic (y/n)?       y
    2.  Please enter operation name:        cosh
    3.  Does your operation change shape?   n
    Please enter operation name:            cosh
    ```
    * Answer the remaining questions as appropriate, and enter 'q' when prompted for parameters<br>


5. Add Custom (cosh) Python Layers:
    1. Copy to the Model Optimizer Ops Directory:<br>
    * This allows MO to find the Python implementation of cosh.<br><br>
    ```
    sudo cp ~/cl_tutorial/OpenVINO-Custom-Layers/2019.r1/cosh.py /opt/intel/openvino/deployment_tools/model_optimizer/mo/ops/
    ```

    2. Copy to Extension Generation Python Target Dir:<br><br>
    ```
    cp ~/cl_tutorial/OpenVINO-Custom-Layers/2019.r1/cosh_ext.py ~/cl_cosh/user_mo_extensions/ops/
    ```

6. Copy CPU and GPU source code to the M.O. extensions directory:<br>
    * This will be used for building a back-end library for applications that implement cosh.<br><br>
    ```
    cp ~/cl_tutorial/OpenVINO-Custom-Layers/2019.r1/ext_cosh.cpp ~/cl_cosh/user_ie_extensions/cpu/
    ```

7. Create the TensorFlow model (weights, graphs, checkpoints):<br>
    * We create a simple model.  The weights are random, and untrained, but this is acceptable for demonstrating Custom Layer conversion.<br><br>
    ```
    cd ~/cl_tutorial/OpenVINO-Custom-Layers/create_tf_model
    ```
    ```
    ./build_cosh_model.py
	```


8. Convert the TensorFlow model to Intel IR format:<br>
    * We run MO for TensorFlow to convert and optimize the new model for OpenVINO. We explicitly set the batch to 1 because the model has an input dim of "-1".   TensorFLow allows "-1" as a "to be filled in later" variable, but MO requires explicit information for the optimization process.  The output is the full name of the final output layer.<br><br>
	```
    cd ~/cl_new
	```
	```
    mo_tf.py --input_meta_graph model.ckpt.meta --batch 1 --output "ModCosh/Activation_8/softmax_output" --extensions ~/cl_cosh/user_mo_extensions --output_dir ~/cl_ext_cosh
	```


9. Compile the C++ extension library:<br>
    * We're building the back-end C++ library to be used by the Inference Engine for executing the cosh layer.<br><br>
    ```
	cd ~/cl_cosh/user_ie_extensions/cpu
    ```
    ```
	cp ~/cl_tutorial/OpenVINO-Custom-Layers/2019.r1/CMakeLists.txt .
    ```
    ```
	mkdir build && cd build
    ```
    ```
	cmake ..
    ```
    ```
	make -j$(nproc)
    ```
    <br>
    
    ```
	cp libcosh_cpu_extension.so ~/cl_ext_cosh/
    ```


10. Test your results:<br>
    Using the C++ Sample
    ```
    ~/inference_engine_samples/intel64/Release/classification_sample -i ~/cl_tutorial/OpenVINO-Custom-Layers/pics/dog.bmp -m ~/cl_ext_cosh/model.ckpt.xml -d CPU -l ~/cl_ext_cosh/libcosh_cpu_extension.so
    ```
    <br>
    Using a Python Sample
    ```
    cp /opt/intel/openvino/deployment_tools/inference_engine/samples/python_samples/classification_sample.py .
    ```
    <br>
    Try running the Python Sample without including the cosh extension library.
    ```
    python3 -i ~/cl_tutorial/OpenVINO-Custom-Layers/pics/dog.bmp -m ~/cl_ext_cosh/model.ckpt.xml -d CPU
    ```
    <br>
    Now run the command with the cosh extension library.
    ```
    python3 -i ~/cl_tutorial/OpenVINO-Custom-Layers/pics/dog.bmp -m ~/cl_ext_cosh/model.ckpt.xml -l ~/cl_ext_cosh/libcosh_cpu_extension.so -d CPU
    ```