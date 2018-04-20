# Managed Zigbee Gateway

This repository contains the software needed to bring up a Managed Zigbee Gateway.

In a realistic IoT deployment of ZigBee products there are three key elements that are required for success. The ZigBee end nodes, a gateway and a management service that assists in the development, deployment and monitoring of the health of the devices at scale. In this app note, we describe how a ZigBee developer can demonstrate a complete IoT system with no new development using a simple vertical stack.

The Zentri Device Management Service (DMS) provides a range of services for managing devices, products and product data. In this demonstration a Zigbee gateway will securely connect with the DMS and authorize it for management. Zigbee end nodes that attach to the gateway are reported to the DMS as end nodes belonging to the gateway. Once the DMS becomes aware of the end nodes it will support tracking of those nodes.

To setup a managed Zigbee gateway you will need at least two [Silicon Labs WSTK and EFR32MG12P radios](https://www.silabs.com/products/development-tools/wireless/mesh-networking/mighty-gecko-starter-kit) that will act as a Zigbee mesh network.

The Zigbee gateway host software is intended to be run on:
* macOS Sierra or later (x86_64)
* Ubuntu 16.04.4 (amd64)

_Note: For Windows users it is recommended to use an Ubuntu 16.04.4 (amd64) Virtual Machine under [Virtual Box](https://www.virtualbox.org/). A preloaded VM image has been provided [here](https://zigbee-managed-gateway.zentri.com/SiliconLabsZigbeeGatewayDMS.ova). The user name and password for this instance is `developer`/`developer`._

## Setup

_Note: Detailed instructions on this setup can be found in [AN1144](https://www.silabs.com/documents/public/application-notes/AN1144.pdf)._

Clone this repo:
```
$ git clone https://github.com/SiliconLabs/managed-zigbee-gateway
$ cd managed-zigbee-gateway
```

_Note: The provided binary images have been generated from the Silicon Labs Gecko SDK 2.2.2 Release and the Zigbee 6.2.3 stack._

### Flash the Devices

Download [Simplicity Commander](https://www.silabs.com/products/mcu/programming-options) and flash the WSTK/EFR32 devices using the firmware images from this repo. More information on Simplicity Commander can be found in [UG162](https://www.silabs.com/documents/public/user-guides/ug162-simplicity-commander-reference-guide.pdf).

_Note: When flashing devices, make sure only one WSTK, the one to be programmed, is connected to the computer._

One device will need to be flashed as an network co-processor (NCP) to act as the gateway to the host computer:
```
$ .</path/to>/commander flash --masserase \
./bin/firmware/ncp-EFR32MG12P432F1024GL125/bootloader-uart-xmodem-combined.s37 \
./bin/firmware/ncp-EFR32MG12P432F1024GL125/ncp-uart-hw.s37
```

One or more devices will need to be flashed as Zigbee end devices:
```
$ <path/to>/commander flash --masserase \
./bin/firmware/ZigbeeEndNode-EFR32MG12P432F1024GL125\
/bootloader-storage-internal-combined.s37 \
./bin/firmware/ZigbeeEndNode-EFR32MG12P432F1024GL125/ZigbeeEndNode.s37
```

### Create a User Account on the DMS

Create a user account for the [Device Management Service](http://dms.zentri.com/signup).

### Provision a new Zigbee Gateway Device on the DMS

Once an account is created, login to the DMS and in the Devices section provision a new software device as type `SOFT`, then download the `certificates.zip` and take note of the UUID and Token provided. For example:
```
UUID  fffffffaa24b7ce88d91a6ccf9e54bfb24e904c2
Token RmwxZ0hhSng2NGxlQkdKbWNCSmZlc0VD
```
This device will be `enabled` until it authenticates with the DMS.

### Deploy the Certificates

The certificates.zip needs to be unzipped into a folder named `certificates` that sits next to the Z3GatewayHost Zigbee gateway executable
```
$ unzip <path/to>/certificates.zip -d ./bin/gateway/<os>/certificates
```

### Start the Zigbee Gateway

Plug in the NCP to the host computer and find the port associated with it. The device will be enumerated as `/dev/tty.usbmodemXXXX` on macOS and on `/dev/ttyACMX` on Ubuntu, where X is some numerical digit.

_Note: If running in a Virtual Machine, the host computer is the VM itself, so don't forget to connect the WSTK device to the VM using the USB connection options before discovering the tty port._

_Note: If running in Ubuntu, the following command must be run before running the gateway:_
```
$ sudo adduser <current user> dialout
```

Run the Zigbee gateway (_Note: If running in Ubuntu, the following command must be prepended by `sudu`_):
```
$ cd bin/gateway/<platform>
$ ./Z3GatewayHost -p /dev/tty<platform specific postfix>
```

The gateway application will post some output, and then a `Z3GatewayHost>` prompt signaling it is ready for interaction.

Form a new Zigbee network:
```
Z3GatewayHost> network leave
Z3GatewayHost> plugin network-creator start 1
```
When this succeeds it will post `EMBER_NETWORK_UP 0x0000`.

Authenticate the gateway with the DMS using the token acquired from the provisioning step:
```
Z3GatewayHost> custom dms-connect "RmwxZ0hhSng2NGxlQkdKbWNCSmZlc0VD"
```

If successful, the device should show up as `active` on the DMS.

Open the network on the gateway to join a device:
```
Z3GatewayHost> plugin network-creator-security open-network
```

### Join a Zigbee End Node to the Gateway

Power up the WSTK acting as a Zigbee end device and Press PB0 to leave any existing network and attempt to join a new one. When the device joins it will show some information about the join and post `Device Joined 0x000B57FFFE491BCD`.

Once this is done, close the network for joining (additionally it should close itself after a timeout period).
```
Z3GatewayHost> plugin network-creator-security close-network
```

The Zigbee gateway device on the DMS should have a matching entry for this device in the Nodes section.
