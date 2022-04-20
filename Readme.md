# Azure Computer Vision - Spatial Analysis Tutorial

In this tutorial, we deploy Azure Spatial Analysis container (module) in Edge device, and handle the tracked events in custom container (custom module).

![tutorial architecture](images/architecture.png?raw=true)

Before starting, please read [here](https://tsmatz.wordpress.com/2019/10/19/azure-iot-hub-iot-edge-module-container-tutorial-with-message-route/) to learn about the basis of IoT Edge deployment and message routing.

## Create IoT Hub resource

Create IoT Hub resource in Azure Portal.

## Set up Edge device

For running Spatial Analysis conatiner, you should prepare a device with NVIDIA GPU 6.0 or above.<br>
Here we use Ubuntu virtual machine (with GPU) for Edge device.

First, create "Ubuntu Server 18.04 LTS" resource with GPU-utilized VM in Azure Portal. In my case, I have used Standard NC6 (NVIDIA Tesla K80) with Gen 1 Ubuntu image.

Next set up IoT Edge runtime with NVIDIA GPU drivers as follows.

### Install GPU driver

Install NVIDIA CUDA driver in this VM (virtual machine).<br>
The simple way to install is to add "NVIDIA GPU Driver Extension" in VM. (Select "Extensions and applications" menu in blade and add this extension.)

> Note : After the installation is completed, please make sure by running ```nvidia-smi```.

### Install docker runtime with nvidia-docker2

Install docker runtime (docker-ce) as follows.

```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
```

After installation, install nvidia-docker2 and restart docker runtime.

```
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update
sudo apt-get install -y docker-ce nvidia-docker2
sudo systemctl restart docker
```

### Install IoT Edge runtime

Install IoT Edge runtime as follows.

```
curl https://packages.microsoft.com/config/ubuntu/18.04/multiarch/prod.list > ./microsoft-prod.list
sudo cp ./microsoft-prod.list /etc/apt/sources.list.d/
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
sudo cp ./microsoft.gpg /etc/apt/trusted.gpg.d/
sudo apt-get update
sudo apt-get install iotedge=1.0.9* libiothsm-std=1.0.9*
```

## Connect Edge device to IoT Hub

Connect your Edge device (Ubuntu VM) to IoT Hub in cloud.

1. Go to Azure Portal and add an IoT Edge device entry in your IoT Hub resource.
2. Get connection string in the generated IoT Edge device entry.
3. Set this connection string in the file /etc/iotedge/config.yaml on Edge device (Ubuntu VM).
4. Restart IoT Edge runtime by running ```sudo systemctl restart iotedge``` in Edge device (Ubuntu VM).

## Set up working client (Ubuntu VM)

For the final setup, please prepare the working client. In this tutorial, we also use Ubuntu VM (Ubuntu Server 18.04 LTS) for this machine. (You can also use Windows or MacOS for working client.)

First, create "Ubuntu Server 18.04 LTS" resource in Azure Portal. (GPU is not needed for this machine.)

Login to this machine and install ```az``` command as follows.

```
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

Next please install Azure IoT extension in ```az```.

```
az extension add --name azure-iot
```

## Create computer vision resource

Create computer vision resource in Azure Portal. Make sure to set "Standard S1" for tier.

After the resouce is created, go to "Keys and Endpoint" menu and copy endpoint and primary key. These are used for Spatial Analysis container.

## Deploy modules

Now deploy modules (containers) in Edge device (Ubuntu VM with GPU) as follows.

Open [deployment.json](./deployment.json), and fill endpoint (see above) in "BILLING" section and fill primary key in "APIKEY" section.<br>
And run the following command in working client (Ubuntu).

```
az iot edge set-modules \
  --hub-name {YOUR IOT HUB NAME} \
  --device-id {YOUR DEVICE ID} \
  --content ./deployment.json
```

After a while, go to IoT Hub resource in Azure Portal. Click IoT Edge device and make sure that all modules are correctly running. (See below.)

![modules](images/modules.png?raw=true)

## Check results

In this deployment, the spatial analysis container will read the stream of [this video](https://teamfileshare.blob.core.windows.net/spatialanalysis-demo-data/line-crossing.mp4?sp=r&st=2021-04-26T22:53:17Z&se=2024-04-27T06:53:17Z&spr=https&sv=2020-02-10&sr=b&sig=sfy4Z%2BQPnMnL2wqA5F0Mw0VVGIoqHG1vtr0IhvhqCuI%3D), instead of RTSP inputs (camera inputs).<br>
By Spatial analysis configuration in deployment.json (see ```SPACEANALYTICS_CONFIG```), the spatial analysis container will track the person's line-crossing events in this video.

After the event is triggered, these events will be passed into Edge Hub in device. (See above architecture.)<br>
These events will then be transferred into testmodule by message routing's setting in deployment.json (see ```routes```).

This testmodule will then outputs the received messages in container logs. (You can see the source code of this module in [here](https://tsmatz.wordpress.com/2019/10/19/azure-iot-hub-iot-edge-module-container-tutorial-with-message-route/).)<br>
You can then see the received messages by running the following command in Edge device (Ubuntu VM).

```
sudo iotedge logs testmodule
```

You can then see the following outputs, which are the messages passed by Spatial Analysis.

```
Received - b'{"events": [{"id": ...
Received - b'{"events": [{"id": ...
Received - b'{"events": [{"id": ...
Received - b'{"events": [{"id": ...
...
```

> Note : To use video recording, make sure to set ```VIDEO_IS_LIVE``` to false in Spatial analysis configuration.

*Tsuyoshi Matsuzaki @ Microsoft*
