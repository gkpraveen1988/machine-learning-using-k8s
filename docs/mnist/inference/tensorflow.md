# Inference of MNIST using TensorFlow on Amazon EKS

This document explains how to perform inference of MNIST model using TensorFlow on Amazon EKS.

## Pre-requisite

1. Create [EKS cluster using GPU](../../eks-gpu.md)
2. Install [Kubeflow](../../kubeflow.md)
3. Basic understanding of [TensorFlow Serving](https://www.tensorflow.org/serving/)

## Upload model

1. A pre-trained model is already available at `mnist/serving/tensorflow/model`. This model requires your serving component has GPU. 

   Use an S3 bucket in your region and upload this model:

   ```
   cd mnist/serving/tensorflow
   aws s3 sync model/ s3://eks-tf-model/mnist/1
   ```

## Install the TensorFlow Serving component

1. Install TensorFlow Serving pkg:

   ```
   ks pkg install kubeflow/tf-serving
   ```

2. Use [Store AWS Credentials in Kubernetes Secret](aws-credential-secret.md) to configure AWS credentials in your Kubernetes cluster. Remember secret name and data fields.

3. Update `modelBasePath` below to match the S3 bucket name where the model is uploaded. Install Tensorflow Serving AWS Component (Deployment + Service):

   ```
   export TF_SERVING_SERVICE=mnist-service
   export TF_SERVING_DEPLOYMENT=mnist

   ks generate tf-serving-service ${TF_SERVING_SERVICE}
   # match your deployment mode name
   ks param set ${TF_SERVING_SERVICE} modelName ${TF_SERVING_DEPLOYMENT}
   # optional, change type to LoadBalancer to expose external IP.
   ks param set ${TF_SERVING_SERVICE} serviceType ClusterIP    

   ks generate tf-serving-deployment-aws ${TF_SERVING_DEPLOYMENT}
   # make sure to match the bucket name used for model
   ks param set ${TF_SERVING_DEPLOYMENT} modelBasePath s3://eks-tf-model/mnist
   ks param set ${TF_SERVING_DEPLOYMENT} s3Enable true
   ks param set ${TF_SERVING_DEPLOYMENT} s3SecretName aws-s3-secret
   ks param set ${TF_SERVING_DEPLOYMENT} s3UseHttps true
   ks param set ${TF_SERVING_DEPLOYMENT} s3VerifySsl true
   ks param set ${TF_SERVING_DEPLOYMENT} s3AwsRegion us-west-2
   ks param set ${TF_SERVING_DEPLOYMENT} s3Endpoint s3.us-west-2.amazonaws.com

   ks param set ${TF_SERVING_DEPLOYMENT} numGpus 1
   ```

4. Deploy Tensorflow Serving components

   ```
   ks apply default -c ${TF_SERVING_SERVICE}
   ks apply default -c ${TF_SERVING_DEPLOYMENT}
   ```

5. Port forward serving endpoint for local testing:

   ```
   kubectl port-forward -n kubeflow `kubectl get pods -n kubeflow --selector=app=mnist -o jsonpath='{.items[0].metadata.name}' --field-selector=status.phase=Running` 8500:8500
   ```

6. Make prediction request. Check sample [mnist_input.json](samples/mnist/serving/tensorflow/mnist_input.json)

   ```
   $ curl -d @mnist_input.json   -X POST http://localhost:8500/v1/models/mnist:predict

   {
    "predictions": [
        {
            "classes": 5,
            "probabilities": [2.34393e-22, 1.37861e-16, 9.06871e-20, 2.48256e-05, 3.94171e-23, 0.999975, 3.54938e-20, 2.2284e-15, 1.07518e-12, 4.44746e-12]
        }
    ]
   }
   ```

   The input is a vector of an image with number 5. The output indicates that the sixth index (starting from 0) has the highest probability.

1. Get serving pod:

   ```
   kubectl get pods -n kubeflow --selector=app=mnist --field-selector=status.phase=Running
   NAME                     READY   STATUS    RESTARTS   AGE
   mnist-7cc4468bc5-wm8kx   1/1     Running   0          2h
   ```

   And then check the logs:

   ```
   kubectl logs mnist-7cc4468bc5-wm8kx -n kubeflow
   ```

## Extract model

### Login to EKS Worker node

1. Get list of the nodes:

   ```
   kubectl get nodes
   NAME                                           STATUS   ROLES    AGE   VERSION
   ip-192-168-40-127.us-west-2.compute.internal   Ready    <none>   10m   v1.11.9
   ip-192-168-72-76.us-west-2.compute.internal    Ready    <none>   10m   v1.11.9
   ```

1. Get IP address of one of the worker nodes:

   ```
   aws ec2 describe-instances \
   --filters Name=private-dns-name,Values=ip-192-168-40-127.us-west-2.compute.internal \
   --query "Reservations[0].Instances[0].PublicDnsName" \
   --output text
   ```

1. Login to worker nodes:

   ```
   ssh -i ~/.ssh/arun-us-west2.pem ec2-user@<worker-ip>
   ```

### Login to Docker container

You can login to the container using [AWS Deep Learning Containers](https://aws.amazon.com/machine-learning/containers/) or `tensorflow/tensorflow` containers. Pick one of the sections below.

#### Using AWS Deep Learning Containers

1. Login to the ECR:

   ```
   $(aws ecr get-login --no-include-email --region us-east-1 --registry-ids 763104351884)
   ```

1. If you have GPU nodes in the cluster:

   ```
   nvidia-docker run -it \
   -v /tmp/saved_model:/model \
   763104351884.dkr.ecr.us-east-1.amazonaws.com/tensorflow-inference:1.13-gpu-py27-cu100-ubuntu16.04 bash 
   ``` 

   If `nvidia-docker` CLI is not available, then use this command:

   ```
   docker run --runtime=nvidia -it \
   -v /tmp/saved_model:/model \
   763104351884.dkr.ecr.us-east-1.amazonaws.com/tensorflow-inference:1.13-gpu-py27-cu100-ubuntu16.04 bash 
   ``` 

1. If you have CPU nodes in the cluster:

   ```
   docker run -it \
   -v /tmp/saved_model:/model \
   763104351884.dkr.ecr.us-east-1.amazonaws.com/tensorflow-inference:1.13-cpu-py27-ubuntu16.04 bash 
   ```

#### Using `tensorflow/tensorflow` containers

1. If you have GPU nodes in the cluster:

   ```
   nvidia-docker run -it \
   -v /tmp/saved_model:/model \
   tensorflow/tensorflow:1.12.0-gpu bash
   ```

   If `nvidia-docker` CLI is not available, then use this command:

   ```
   docker run --runtime=nvidia -it \
   -v /tmp/saved_model:/model \
   tensorflow/tensorflow:1.12.0-gpu bash
   ```

1. If you have CPU nodes in the cluster:

   ```
   docker run -it \
   -v /tmp/saved_model:/model \
   tensorflow/tensorflow:1.12.0 bash
   ```

### Export model

1. Export your model:

   ```
   apt update && apt install git
   git clone -b r1.12.0 https://github.com/tensorflow/models.git /tmp/models

   export PYTHONPATH="$PYTHONPATH:/tmp/models"
   pip install --user -r /tmp/models/official/requirements.txt

   python /tmp/models/official/mnist/mnist.py --export_dir /model/
   exit
   ```

   Now we get the model in Tensorflow's `SavedModel` format in `/tmp/saved_model` on the host.

   ```
   |-- saved_model.pb
   `-- variables
       |-- variables.data-00000-of-00001
       `-- variables.index
   ```
