# Azure Computer Vision - Spatial Analysis Tutorial

In this tutorial, we deploy Azure Spatial Analysis container (module) in Edge device, and handle the captured events in custom container (custom module).

![tutorial architecture](images/architecture.png?raw=true)

Before starting, please read [here](https://tsmatz.wordpress.com/2019/10/19/azure-iot-hub-iot-edge-module-container-tutorial-with-message-route/) to learn about the basis of IoT Edge deployment and message routing.

## 1. Create IoT Hub resource

Create IoT Hub resource in Azure Portal.

## 2. Create and set up Edge device (Ubuntu VM)

For running Spatial Analysis conatiner, you should prepare a device with NVIDIA GPU 6.0 or above.<br>
In this example, we use Ubuntu virtual machine (with GPU) for Edge device.

First, create "Ubuntu Server 18.04 LTS" resource with GPU-utilized instance in Azure Portal. In my case, I have created a Standard NC6 (NVIDIA Tesla K80) VM with Gen 1 Ubuntu image.

Next, install and set up IoT Edge runtime with NVIDIA GPU drivers as follows.

### Install GPU driver

Install NVIDIA CUDA driver in this VM (Edge device).<br>
The simple way to install GPU drivers is to add "NVIDIA GPU Driver Extension" for VM in Azure Portal. (Select "Extensions and applications" menu in blade and then add this extension.)

> Note : After the driver's installation is completed, please check if the driver is enabled by running ```nvidia-smi``` command.

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

Next, install nvidia-docker2 and restart docker runtime.

```
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update
sudo apt-get install -y docker-ce nvidia-docker2
sudo systemctl restart docker
```

### Install IoT Edge runtime in Edge device

Install IoT Edge runtime as follows.

```
curl https://packages.microsoft.com/config/ubuntu/18.04/multiarch/prod.list > ./microsoft-prod.list
sudo cp ./microsoft-prod.list /etc/apt/sources.list.d/
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
sudo cp ./microsoft.gpg /etc/apt/trusted.gpg.d/
sudo apt-get update
sudo apt-get install iotedge=1.0.9* libiothsm-std=1.0.9*
```

## 3. Connect your Edge device to IoT Hub

Connect your Edge device (Ubuntu VM) to IoT Hub in cloud as follows.

1. Go to Azure Portal and go to IoT Hub resource. Click "IoT Edge" menu and add an IoT Edge device (Edge-enabled device) entry in your IoT Hub resource.
2. Get connection string in the generated IoT Edge device entry.
3. Login to Edge device (Ubuntu VM) and set this connection string in the file ```/etc/iotedge/config.yaml```.
4. Restart IoT Edge runtime by running<br>
```sudo systemctl restart iotedge```<br>
in Edge device (Ubuntu VM).

## 4. Create and set up working client (Ubuntu VM)

For the final setup, please provision your working client.<br>
You can also use Windows or MacOS for working client, but in this tutorial, we also use Ubuntu VM (Ubuntu Server 18.04 LTS) for the client.

First, create "Ubuntu Server 18.04 LTS" resource in Azure Portal. (GPU is not needed for this client's machine.)

Login to this machine (client) and install ```az``` command as follows.

```
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

Next please install Azure IoT extension in ```az```.

```
az extension add --name azure-iot
```

## 5. Create computer vision resource

Create computer vision resource in Azure Portal. Make sure to set "Standard S1" in pricing tier.

After the resouce is created, click "Keys and Endpoint" menu and then copy endpoint and primary key. (These are used for Spatial Analysis container deployment.)

![cognitive services endpoint and key](images/endpoint_and_key.png?raw=true)

## 6. Deploy modules

Now let's deploy modules (containers) in your Edge device (GPU-utilized Ubuntu VM) as follows.

Before deployment, open [deployment.json](./deployment.json), and fill the endpoint address for computer vision resource (see above) in "```BILLING```" environment on spatial analysis module. Also, fill primary key (see above) in "```APIKEY```" environment.

```
{
  "modulesContent": {
    "$edgeAgent": {
      "properties.desired": {
        ...

        "modules": {
          "spatialanalysis": {
            "settings": { ... },
            "type": "docker",
            "env": {
              ...

              "BILLING":{
                "value": "<Use endpoint from Computer Vision Azure resource>"
              },
              "APIKEY":{
                "value": "<Use the key from Computer Vision Azure resource>"
              },
              ...

            }
          },
          ...

        }
      }
    },
    ...

  }
}
```

To deploy modules in your Edge device (Ubuntu VM) thru IoT Hub, copy ```deployment.json``` in your working client and run the following command. (Replace the following ```{YOUR IOT HUB NAME}``` and ```{YOUR DEVICE ID}``` with yours.)

```
az iot edge set-modules \
  --hub-name {YOUR IOT HUB NAME} \
  --device-id {YOUR DEVICE ID} \
  --content ./deployment.json
```

After a while, go to IoT Hub resource in Azure Portal and click "IoT Edge" menu.<br>
Click and see your IoT Edge device, and make sure that all modules are correctly running.<br>
(It will take tens of minutes for the installation of spatial analysis container.)

![modules](images/modules.png?raw=true)

## 7. Check events

In this tutorial, the spatial analysis container (module) will read the stream of [this video file](https://teamfileshare.blob.core.windows.net/spatialanalysis-demo-data/line-crossing.mp4?sp=r&st=2021-04-26T22:53:17Z&se=2024-04-27T06:53:17Z&spr=https&sv=2020-02-10&sr=b&sig=sfy4Z%2BQPnMnL2wqA5F0Mw0VVGIoqHG1vtr0IhvhqCuI%3D) (which is also used in Spatial Analysis tutorial in official document), instead of RTSP endpoint (camera inputs).<br>
By Spatial analysis configuration in deployment.json (see ```SPACEANALYTICS_CONFIG``` in [deployment.json](./deployment.json)), the spatial analysis container will capture the person's line-crossing events in this video.

> Note : To use video recording in Spatial Analysis, make sure to set ```VIDEO_IS_LIVE``` to false in configuration.

After the event is triggered, these events will be passed into Edge Hub in device. (See above illustrated architecture.)<br>
These events will then be transferred into testmodule (custom module) by message routing's setting (see ```routes```) in deployment.json.

This testmodule will then just output the received messages in container logs. (You can see the source code of this module in [here](https://tsmatz.wordpress.com/2019/10/19/azure-iot-hub-iot-edge-module-container-tutorial-with-message-route/).)<br>
You can then see the received messages by running the following command in Edge device (Ubuntu VM).

```
sudo iotedge logs testmodule
```

The following outputs (which are messages passed by Spatial Analysis) will be shown.

```
Received - b'{"events": [{"id": ...
Received - b'{"events": [{"id": ...
Received - b'{"events": [{"id": ...
Received - b'{"events": [{"id": ...
...
```

Please change video file or configuration in deployment.json, and see how message is changed.

## 8. [Optional] Debug with X Display

For the debugging purpose, you can also see how it detects with Spatial analysis visualizer. (See below.) With this visualizer, you can check whether your coordinate's configuration fits to the captured screen.

![visualizer](images/visualizer.png?raw=true)

In the last part of this tutorial, I'll show you how to enable this visualizer.<br>
In this example, we are using Ubuntu Server 18.04 LTS for IoT Edge device. To simplify configuration, here we use X display with this host and remote desktop client in Windows.

In order to install X remote desktop components, login to your Edge device (GPU-utilized Ubuntu) and run the following commands.<br>

```
sudo apt-get install -y lxde
sudo apt-get install -y xrdp
```

Now, start RDP service as follows. (You should fill the password for start daemon.)<br>
After this setting, restart your Edge device computer (Ubuntu).

```
/etc/init.d/xrdp start  # password is required
```

After the computer (Ubuntu) is restarted, open port 3389 and connect to this computer with remote desktop client in your local Windows.<br>
To get display id, open terminal (such as LXTerminal) and run as follows.

```
env | grep DISPLAY
```

Next, run the following command to make remote X clients to connect this device from any host.

```
xhost +
```

Same as you have changed in deployment.json, open [deployment_debug.json](./deployment_debug.json) and fill "```BILLING```" and "```APIKEY```" environments in spatial analysis configuration.<br>
Futhermore, please fill X display id (which is copied in above) in ```DISPLAY``` environment setting, as well.

```
{
  "modulesContent": {
    "$edgeAgent": {
      "properties.desired": {
        ...

        "modules": {
          "spatialanalysis": {
            "settings": { ... },
            "type": "docker",
            "env": {
              "DISPLAY": {
                "value": "<Fill display id>"
              },
              ...

              "BILLING":{
                "value": "<Use endpoint from Computer Vision Azure resource>"
              },
              "APIKEY":{
                "value": "<Use the key from Computer Vision Azure resource>"
              },
              ...

            }
          },
          ...

        }
      }
    },
    ...

  }
}
```

In this new deployment configuration (deployment_debug.json), the following 3 settings are changed for visualization and debugging.

- Set your X remote display id in ```DISPLAY``` environment. (See above.)
- Set debug version of operation id (```cognitiveservices.vision.spatialanalysis.debug```) in ```operationId``` entry.
- Added the following entry in spatial analysis configuration.<br>
  ```"VISUALIZER_NODE_CONFIG": "{ \"show_debug_video\": true }"```

Now, login to your working client (Ubuntu) and deploy modules with this new deployment configuration (deployment_debug.json).

```
az iot edge set-modules \
  --hub-name {YOUR IOT HUB NAME} \
  --device-id {YOUR DEVICE ID} \
  --content ./deployment_debug.json
```

In your remote desktop client (logging in Edge device's host), copy ```.Xauthority``` into spatial analysis container.

```
sudo docker cp .Xauthority spatialanalysis:/root/.Xauthority
```

Now let's restart spatial analysis container.

```
sudo docker stop spatialanalysis
sudo docker start spatialanalysis
```

When you restart container, the visualizer will launch in your remote X display.<br>

![visualizer](images/visualizer.png?raw=true)

> Note : You can also use diagnostics and telegraf containers in Spatial Analysis for logging and monitoring in Azure Monitor.

*Tsuyoshi Matsuzaki @ Microsoft*
