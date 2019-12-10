#  Device Core

Everything contained in this repo is together referred to as "device core," it  contains setup scripts, monitoring scripts, and an HTTP server that is used to facilitate device actions to nimbleNODE units or converted units.  A key component of this is a publicly accessible API, which can be used to commission WiFi and shell scripts that establish a device hotspot.

This documentation will discuss a full breakdown of everything contained on a device acting as a nimbleNODE, as well as provide a detailed run down of each publicly accessible API endpoint.

### How to make a device a nimbleNODE? 

At the core of each device that will become a nimbleNODE is a linux distribution (typically Ubuntu) with the contents of this repo and the following dependencies installed: 

- NVM (Node Version Manager) - used to install NodeJS
- NodeJS (version 9) - used to run the HTTP servers
- PM2 - a daemon used to manage the HTTP server and scripts
- dnsmasq - used to perform DHCP from devices connected to our hotspot
- hostapd - allows the device to create a hotspot for others to connect

```shell
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # loads bash_completion
nvm install v9

sudo ln -s "$NVM_DIR/versions/node/$(nvm version)/bin/node" "/usr/local/bin/node"
sudo ln -s "$NVM_DIR/versions/node/$(nvm version)/bin/npm" "/usr/local/bin/npm"
sudo apt-get update
sudo apt-get install dnsmasq hostapd zip unzip -y
```

**Recommended:**  To install the necessary dependencies, use the script `scripts/device/nimble-init-setup.sh`. 

With the dependencies installed, proceed to the user directory of the device and carry out each of the following steps to properly configure your device as a nimbleNODE: 

```shell
cd ~
git clone https://github.com/dondreytaylor/nimble-device-core.git
cd nimble-device-core
npm i
pm2 start pm2.config.js # Starts Scripts and HTTP Server
```

**Note:** If pm2.config.js gets updated, run `pm2 reload pm2.config.js` followed by `pm2 save` to ensure that your changes remain after reload.

### INCREASING MEMORY PAGING

In the event you would like to increase paging to compensate for low memory devices like the raspberry pi, edit `/etc/dphys-swapfile` to increase paging and restart dphys with the command `/etc/init.d/dphys-swapfile restart`.

### DEVICE SCRIPTS

Within this repo there are 3 scripts used for device setup. These scripts can be found within the `scripts/device` directory.  Two of the scripts are used to actively monitor configurations while the device is on and the other script is used to perform the initial setup.  

* `nimble-builddata-checks.sh` - *(gets run by [pm2.config.js](https://github.com/dondreytaylor/nimble-device-core/blob/master/pm2.config.js))* used to monitor builddata directory.  The builddata directory is the folder containing the data directory for specific chain associated with the device.  For example, in the case of bitcoin, this directory would be the contents of datadir. This script works by checking to see if the builddata does not exist or that the device has been improperly shutoff every 3 seconds. In the case of an improper shutdown or the deletion of the builddata directory, the script will begin rebuilding the builddata directory by unzipping builddata.zip which is required to exist in the same directory as was builddata. Depending on the processing power and RAM of the device, unzipping builddata can take anywhere from 15 to 25 minutes. 
* `nimble-checks.sh` - *(gets run by [pm2.config.js](https://github.com/dondreytaylor/nimble-device-core/blob/master/pm2.config.js))* used to monitor the hotspot configuration. To ensure that the device has the correct configurations to broadcast as a hotspot, this script will check to make sure that the hotspot network rules and hotspot configuration have been updated; the script does this by checking for the existence of the macaddress file which gets created by this script on the first run. 
* `nimble-init-setup.sh` - (gets manually run) This script automates the installation of the dependencies (described in the prior section) and the configurations (setup by nimble-checks.sh).

### SERVICE SCRIPTS

Scripts that need to be run prior to shutdown and on startup, can be found within the `services/startup` directory and must be properly setup as services within the system. To setup these files as services, copy them to `/etc/systemd/system/` and then run `sudo systemctl daemon-reload` 

* `powercheck.service` - this service executes a script called  [powercheck.sh](https://github.com/dondreytaylor/nimble-device-core/blob/master/scripts/services/startup/powercheck.sh), which creates and deletes the shutoff indication file named `improper_shutdown_detection_flag`, used to determine whether the device was shutdown cleanly. 

### DEVICE REQUESTS

To interact with your device, an HTTP server exposes a variety of endpoints that you can use. There are two ways that you can make calls to the api: the first is by connecting to your nimbleNODE directly via the hotspot, and the second is by routing your requests through the relay server. 

**Sending API requests directly to the device via hotspot** 

To send http calls to your device, you'll need to first connect to its hotspot. With the device powered on, choose the SSID`nimblenode-<alpha numeric>` or the SSID that you've assigned to your device. The default password is the same as the SSID if you have not yet changed it. 

With your computer connected to your device, type in the following into a web browser `http://192.168.10.1:3001`. If successfully connected, you should see a welcome JSON response.

```json
{
   "message": "Welcome to the Nimble device API",
   "time": 1575747025793
}
```

**Note:** ``http://192.168.10.1:3001`` is the endpoint that you can use to route all of your requests to your device. 

**Sending API and CLI requests through a relay server** 

In addition to sending requests to your device when connected to its hotspot, you can also route encrypted messages to your device by using a relay server. Currently nimblenode.io provides a free relay server service, however in the near future the relay server will be made fully open source.  

To send requests through the relay server provided by nimblenode.io, you'll need to do a few things: connect to your device directly in order to retrieve a public key that can be used to encrypt messages and MAC address, commission wifi on your device, and lastly prepare your message and send it to the relay server endpoint that will pass it to your device

* **Step 1: Connect to your device** 

  Connect to your device directly by powering it up and choosing the SSID from the list of available networks. 

* **Step 2:  Encryption Key** 

  First generate a public key using the route `//192.168.10.1:3001/api/encryption/set_device_keys` and make note of both the `keyid` and `publickey`. The keyid will be used to let the device know two things: which private key to use when decrypting and which public key to use when encrypting a response. 

  ```javascript
  {
     "key": {
  	"keyid": "0abcf62784f21cbf18955bc551dbb315",
  	"publickey": "-----BEGIN PUBLIC KEY----- MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBAMdkml0yZ6TLSWOmCUR0c8FBMa6FCaho N4TOGU+1tCSYKCTuZHlzB9MjWRo8Nw/1yblfOIpbEThRPCaKlcMDBQ8CAwEAAQ== -----END PUBLIC KEY-----"
     }
  }
  ```

  Next, we'll need to generate a public key that the device can use to encrypt a response. Send a POST request to the route `//192.168.10.1:3001/api/encryption/set_remote_keys` with the same keyid that you retrieved when generating a public/private key pair for the device, and a new public. It is important to note that you should store the private key for your new public/private key pair and the keyid somewhere safe. The request to `/api/encryption/set_remote_keys` should look like this (where the public key is some unique key): 

  ```json
  {
    "keyid": "0abcf62784f21cbf18955bc551dbb315",
    "publickey": "your-new-public-key-here"
  }
  ```

  If your key is set successfully you'll get the response `{"saved": true}`.

* **Step 3: Commission Wifi** 

  To commission wifi, provide an SSID and password to this route `//192.168.10.1:3001/api/wifi/network/new` as GET paramters in the following way: `//192.168.10.1:3001/api/wifi/network/new?ssid=MySSID&password=MyPASSWORD&should_save=true`, replace MySSID and MyPASSWORD with the ssid and password for your network.  To scan for nearby networks use the route `/api/wifi/scan` or `/api/wifi/scan/essid`. Connecting to your network may take up to 1 minute and 30 seconds. Use the route `/api/wifi/status` or `/api/wifi/inet` to determine if your device has been successfully connected to wifi. 

* **Step 4: Find your MAC address** 

  Now that you're device is connected, it's time to get the Mac address. The Mac address helps with routing requests through the relayer. To get the mac address from your device do a GET request to the endpoint `//192.168.10.1:3001/api/device/macaddress` and make note of the mac address that gets returned.

  ```json
  {
    "macaddress": "b2:37:eb:5e:c5:9e"
  }
  ```

* **Step 5: Preparing request** 

  There are two types of requests that you can send to your device: a CLI command, or an API command. CLI commands get carried out by the CLI of the chain on your device. For example, if you are running Bitcoin, these commands will get passed to `bitcoin-cli`. API commands on the other hand get routed to API endpoints that are otherwise only available when you directly connect to your device's hotspot (e.g. checking network status, device details, connecting to wifi, etc).

* **Step 6: Sending an API Command**

  Now that you have your Mac address and public key handy, it's time to prepare your request. Before preparing the body of your request, you'll need to prepare a JSON string that is in the format:  `{"namespace": "", "method": "", "parameters": []}`. 

  The `namespace` value maps to the object within the API library (found within [api.js](https://github.com/dondreytaylor/nimble-device-core/blob/master/lib/api.js)). For example, accessing the wifi functions would require that you set `wifi` as your namespace. The `method` value is the function name found within the namespace object. For example, checking the wifi status means setting `status` as our method name. Lastly, the `parameters` value is a list of parameters to be passed to the function of the method specified. For example, if you we would like to call the `remote_encrypt_key` method on the `encryption` namespace, then we would need to set pass a keyid parameter like this `"parameters": ['keyid-goes-here']`. It is important to note that the order of the parameters must match the method signature of the function being called.

  With the JSON string prepared, it's time to construct the final payload for our request to the relay server. The payload will contain your mac address, your JSON string encrypted, and your keyid. The relay server payload will look like this: 

  ```json
  {
    "deviceid": "b2:37:eb:5e:c5:9e",
    "command": "JSON string encrypted goes here",
    "keyid": "61ba036fab9033f5f255cd089b1ba28c"	
  }
  ```

  Here is an example request with an encrypted payload. 

  ```json
  {
    "deviceid": "b2:37:eb:5e:c5:9e",
    "command": "Id7aW+1NOG0Ism6aLVVFjUE1sEl3lJNy82VVfs7KxwKR3gjt7M8rfdwHNLhhLPVP00P2w1VWViqRei+E/PoJYkkdpN8FSwA72aUSb8Zxsr1x8inU9XMyXf/Vf/QSjjF68qxedOBjXNh1rpP+sFMV+E21Y+vlHzrHVPGNjd+GjgdWEIr1pxZEcQeTJqZKwaOXZNYu5ZWh3YOdtO0a3Datpje8h0qE9nnhhAtXBBOxxzYFs1Z3R6Hdsg2OhbtPW2+G",
    "keyid": "0abcf62784f21cbf18955bc551dbb315"
  }
  ```

  The `deviceid` is your device mac address, this tells the relay server which device the command is for. The `command` is your JSON string encrypted using the public key that you retrieved from your device. To encrypt your JSON string refer to the [encryption_tests.js](https://github.com/dondreytaylor/nimble-device-core/blob/master/encryption_tests.js). The `keyid` is the key id for the public key that you would like the device to  encrypt the response with. Refer to the API for how to associate a publc key with a key id on a device.

  It's now time to send your payload to the relay server. Send your request as a POST to `http://relay.nimblenode.io:3000/api/device/command`.  If successfully added, the response you will get back is the following response. 

  ```json
  { "added": true, "commandid": "77acb2e4f903e717b1dab8a37a2d59a2" }
  ```

  Make note of the `commandid` as this will be used to retrieve the response from the relay server. At this point, your request will remain queued until your device checks for new commands to run.  

  To check for a command response, send a GET request to the relayer endpoint along with your `deviceid` and the corresponding `commandid` like so `http://relay.nimblenode.io:3000/api/device/command/response?deviceid=b2:37:eb:5e:c5:9e&commandid=77acb2e4f903e717b1dab8a37a2d59a2`. This will produce an output in the following format: 

  ```json
  {
     "commandid": "77acb2e4f903e717b1dab8a37a2d59a2",
     "command": "encrypted-command-sent-here",
     "commandresponse": "encrypted-response-here",
     "keyid": "0abcf62784f21cbf18955bc551dbb315",
     "added": 1575922212960,
     "added_formatted": "Mon Dec 09 2019 20:10:12 GMT+0000 (UTC)"
  }
  ```

  Within the response you'll find a few different values. The `commandid` is the ID that is associated with the original command sent. `command` is the original command encrypted using the public key associated with the `keyid` on the device. `commandresponse` is the encrypted response for the command run by the device, to decrypt the response, use the private key that you mapped to `keyid` of the device. 

* **Step 7: Sending a CLI Command**

  Sending a command to the CLI on the device is similar to sending commands to the API. The CLI command payload will have the same values except instead of `command` we will pass `clicommand` like so:

  ```json
  {
    "deviceid": "b2:37:eb:5e:c5:9e",
    "clicommand": "encrypted-command-here",
    "keyid": "0abcf62784f21cbf18955bc551dbb315"
  }
  ```

  The `clicommand` will follow the format `{"method": "", "parameters": ""}`. `method` will contain the CLI command that you would like to call on the daemon. `parameters` will contain the arguments passed to the CLI command. For example, if we want to perform a `sendmany` we would  prepare our JSON string like so:

  ```json
  {
     "method": "sendmany", 
     "parameters": "'{\"BFTV27j1GnHPEEXNxbkRN6QgD4QHw5gRew\": 10}'"
  }
  ```

  **Important Note:** To send a CLI command, the daemon on your device must be running. Refer to the API documentation in the later section below, on how to start the daemon and check its status.

  Here is an example of sending an encrypted CLI command to the relay server.

  ```json 
  {
    "deviceid": "b2:37:eb:5e:c5:9e",
    "clicommand": "ibeWKMiAl7yUERIgLRB/frLFL1ZS0f4X5lvcqjDsotu2E5mB5S32DXv2iNVHxFy+RdbsGFV63VnmDDrYAdbxBkp7DXxhzTzllgfzQsVjxooJdKvfX8HBKO6HOyhXdrxQiByGXsRHnMUqMuKLIAhvd/75mR+DCfdGYxsKxlkoIDk=",
    "keyid": "0abcf62784f21cbf18955bc551dbb315"
  } 
  ```

  If your command was sent successfully, it should produce a response that looks like this:

  ```json 
  {
    "added": true,
    "cliid": "7314f8f9b7572325beedfc141eefeaa1"
  }
  ```

  Save the `cliid` as you will need it later when you retrieve the response from the relay server for this request. With the `cliid` in saved, we'll now construct a GET request that we can send to the relay server, containing boththe `cliid` and our `deviceid` as parameters: `http://relay.nimblenode.io:3000/api/device/cli/response?deviceid=b2:37:eb:5e:c5:9e&cliid=7314f8f9b7572325beedfc141eefeaa1`. The response for your request should look like this:

  ```json 
  {
    "cliid": "7314f8f9b7572325beedfc141eefeaa1",
    "clicommand": "cli-command-encrypted-here",
    "cliresponse": "cli-response-encrypted-here",
    "keyid": "0abcf62784f21cbf18955bc551dbb315",
    "added": 1575933043531,
    "added_formatted": "Mon Dec 09 2019 23:10:43 GMT+0000 (UTC)"
  }
  ```

  Within the response you'll find a few different values. The `cliid` is the ID that is associated with the original cli command sent. `clicommand` is the original CLI command encrypted using the public key associated with the `keyid` on the device. `cliresponse` is the encrypted response for the CLI command run by the device, to decrypt the response, use the private key that you mapped to `keyid` of the device. 

### API DOCUMENTATION

`[GET]` **/api/encryption/keys**

Retrieves both public and private keys stored on the device along with their corresponding  keyid. The list consists of a mixture of device and remote keys. Device keys contain a public key used to encrypt data by other devices, and a private key used to decrypt data sent to the device by other devices. Remote keys on the other hand, are public keys that should be used to encrypt data for other devices that specify a keyid. 

> Namespace:  **encryption**
>
> Response: 
>
> ```json 
> {
>   "keys": {
>    	 "key-347879b5e992b06fd1bfeecb4477f87c-device": {
> 			 "keyid": "347879b5e992b06fd1bfeecb4477f87c",
> 			 "publickey": "-- public key here --",
> 			 "privatekey": "-- private key here --"
> 	    },
>    	 "key-347879b5e992b06fd1bfeecb4477f87c-remote": {
> 			 "keyid": "347879b5e992b06fd1bfeecb4477f87c",
> 			 "publickey": "-- public key here --"
> 	    }
>   }
> }
> ```
>
> 

`[GET]` **/api/encryption/set_device_keys**

Generates a public and private key pair for a newly created keyid.  The private key is not revealed.

> Namespace:  **encryption**
>
> Response: 
>
> ```json 
> {
>    "key": {
>       "keyid": "8cbfe50c38fba9c563954a9f9c0a16ee",
>       "publickey": "-- public key --"
>    }
> }
> ```
>
> 

`[GET]` **/api/encryption/remove_keys**

Deletes all private and public keys stored on the device, this includes both device and remote keys.

> Namespace:  **encryption**
>
> Response: 
>
> ```json 
> {"removed": true}
> ```
>
> 

`[POST]` **/api/encryption/set_remote_keys**

Associates a keyid with a public key for the device. The public key will be used to encrypt data for a device that specifies this keyid within a request.

> Namespace:  **encryption**
>
> Parameters: 
>
> * **publickey** - The public key to be used to encrypt data by the device for another device with the associated private key.
> * **keyid** - A unique identifier to associate the public key to. 
>
> Response (Success): 
>
> ```json 
> {"saved": true}
> ```
>
> Response (Failure)
>
> ```json 
> {"message":"", "error": true, "saved": false}
> ```
>
> 

`[POST]` **/api/encryption/device_encrypt_key**

Retrieve a device public key to be used to encrypt data for this device given a keyid. 

> Namespace:  **encryption**
>
> Parameters: 
>
> * **keyid** - A unique identifier associated with public key you would like to retrieve.
>
> Response: 
>
> ```json 
> {
>    "keys": {
>       "keyid": "8cbfe50c38fba9c563954a9f9c0a16ee",
>       "publickey": "-- public key --"
>    }
> }
> ```
>
> 

`[POST]` **/api/encryption/remote_encrypt_key**

Retrieve a remote public key to be used to encrypt data for another device given a keyid. 

> Namespace:  **encryption**
>
> Parameters: 
>
> * **keyid** - A unique identifier associated with public key you would like to retrieve.
>
> Response: 
>
> ```json 
> {
>    "keys": {
>       "keyid": "8cbfe50c38fba9c563954a9f9c0a16ee",
>       "publickey": "-- public key --"
>    }
> }
> ```
>
> 

`[POST]` **/api/encryption/update_remote_encrypt_key**

Updates a remote public key given a keyid. 

> Namespace:  **encryption**
>
> Parameters: 
>
> * **publickey** - The new public key you would like to set as your remote key
>
> * **keyid** - A unique identifier associated with public key you would like to update.
>
> Response (Success): 
>
> ```json 
> {"updated": true}
> ```
>
> Response (Failure):
>
> ```json 
> {"message": "", "error": true}
> ```
>
> 

`[GET]` **/api/wifi/status**

Retreives current status of device wifi.

> Namespace:  **wifi**
>
> Response (Example): 
>
> ```json 
> {
>   "status": {
>      "bssid": "14:b2:03:6a:7c:33",
>      "frequency": 2412,
>      "mode": "station",
>      "key_mgmt": "wpa2-psk",
>      "ssid": "Network-2.4",
>      "pairwise_cipher": "CCMP",
>      "group_cipher": "TKIP",
>      "p2p_device_address": "b5:ac:1b:e8:4f:14",
>      "wpa_state": "COMPLETED",
>      "ip": "10.1.10.154",
>      "mac": "b3:25:eb:8e:c4:1e",
>      "uuid": "61b5ef2d-6c19-5418-9228-72e3306c658f",
>      "id": 1
>   },
>   "err": null
> }
> ```
>
> 

`[GET]` **/api/wifi/inet**

Retreives current network status of device. If the device is connected to wifi, this call will return the local IP address assigned to it. 

> Namespace:  **wifi**
>
> Response (Example): 
>
> ```json 
> {
>    "inets": " inet 10.1.10.154 netmask 255.255.255.0 broadcast 10.1.10.255 "
> }
> ```
>
> 

`[GET]` **/api/wifi/save/config**

Saves the configuration of wpa supplicant

> Namespace:  **wifi**
>
> Response: 
>
> ```json 
> {"saved": true}
> ```
>
> 

`[GET]` **/api/wifi/scan**

Triggers a scan of nearby wifi networks.

> Namespace:  **wifi**
>
> Response: 
>
> ```json 
> {
>    "networks": [
>       {
>          "address": "51:b2:03:6a:7c:18",
>          "channel": 1, 
>          "frequency": 2.412,
>          "mode": "master",
>          "quality": 67,
>          "signal": -43,
>          "ssid": "Network-2.4",
>          "security": "wpa2"
>       },
>       {
>          "address": "52:b2:03:6a:7c:48",
>          "channel": 1, 
>          "frequency": 2.412,
>          "mode": "master",
>          "quality": 67,
>          "signal": -43,
>          "ssid": "MyNetwork",
>          "security": "wpa2"
>       }
>   ],
>   "err": null
> }
> ```
>
> 

`[GET]` **/api/wifi/scan/essid**

Triggers a scan of nearby wifi networks. This is an alternative route for retrieving wifi networks should the /api/wifi/scan route only return a single network when the device is already connected to an existing network. The `hasencryption` value returned in the response pertains to whether or not the network is WPA password protected. 

> Namespace:  **wifi**
>
> Response: 
>
> ```json 
> {
>    "networks": [
>      {
>          "ssid": "Network-2.4",
>          "hasencryption": true
>      },
>      {
>          "ssid": "MyNetwork",
>          "hasencryption": false
>      }
>   ],
>   "err": null
> }
> ```
>
> 

`[GET]` **/api/wifi/network/new**

Connects the device to a nearby network given an `ssid` and `password`.

> Namespace:  **wifi**
>
> Parameters: 
>
> * **ssid** - the name of the network that you would like to connect to. This must match a network name provided by the wifi scan route. 
> * **password** - the wpa password associated with the network you would like to connect to. 
> * **should_save** - saves the network to the wpa supplicant file. 
>
> Response (Example Success): 
>
> ```json 
> {"added": true, "networkid": 1}
> ```
>
> Response (Failure): 
>
> ```json 
> {"added": false}
> ```
>
> 

`[GET]` **/api/wifi/network/existing**

Changes network of device to a different network given its `networkid`. 

> Namespace:  **wifi**
>
> Parameters: 
>
> * **networkid** - the ID of the network within the wpa supplicant file.
>
> Response (Success): 
>
> ```json 
> {"selected": true}
> ```
>
> Response (Failure): 
>
> ```json 
> {"selected": false}
> ```
>
> 

`[GET]` **/api/wifi/network/remove**

Removes a saved network from the device.

> Namespace:  **wifi**
>
> Parameters: 
>
> * **networkid** - the ID of the network within the wpa supplicant file.
>
> Response (Success): 
>
> ```json 
> {"removed": true}
> ```
>
> Response (Failure): 
>
> ```json 
> {"removed": false}
> ```
>
> 

`[GET]` **/api/wifi/network/reconnect**

Reconnects device to an available network within the wpa supplicant file.

> Namespace:  **wifi**
>
> Response (Success): 
>
> ```json 
> {"reconnected": true}
> ```
>
> Response (Failure): 
>
> ```json 
> {"reconnected": false}
> ```
>
> 

`[GET]` **/api/wifi/network/disconnect**

Disconnects device from current wifi network.

> Namespace:  **wifi**
>
> Response (Success): 
>
> ```json 
> {"disconnected": true}
> ```
>
> Response (Failure): 
>
> ```json 
> {"disconnected": false}
> ```
>
> 

`[GET]` **/api/wifi/networks/saved**

Lists networks saved on the device/wpa supplicant file.

> Namespace:  **wifi**
>
> Response: 
>
> ```json 
> {
>   "networks": [
>     {
>       "networkid": "0",
>       "ssid": "Network-2.4",
>       "bssid": "any",
>       "flags": ""
>     },
>     {
>       "networkid": "1",
>       "ssid": "MyNetwork",
>       "bssid": "any",
>       "flags": "[CURRENT]"
>     }
>   ]
> }
> ```
>
> 

`[GET]` **/api/device/reboot**

Triggers a device reboot.

> Namespace:  **device**
>
> Response: 
>
> ```json 
> {"reboot": true}
> ```
>
> 

`[GET]` **/api/device/shutdown**

Triggers a device shutdown.

> Namespace:  **device**
>
> Response: 
>
> ```json 
> {"shutdown": true}
> ```
>
> 

`[GET]` **/api/device/details**

Current details of device network, disk, cpu usage, etc.

> Namespace:  **device**
>
> Response: 
>
> ```json 
> {
>   "load": {
>     "avgload": 0.04,
>     "currentload": 6.342375360990917,
>     "currentload_user": 4.487036934384161,
>     "currentload_system": 1.8553166883505772,
>     "currentload_nice": 0.000021738256178827593,
>     "currentload_idle": 93.65762463900909,
>     "currentload_irq": 0,
>     "raw_currentload": 29176100,
>     "raw_currentload_user": 20641200,
>     "raw_currentload_system": 8534800,
>     "raw_currentload_nice": 100,
>     "raw_currentload_idle": 430842400,
>     "raw_currentload_irq": 0,
>     "cpus": [
>       {
>         "load": 5.49883311343462,
>         "load_user": 3.711466163775449,
>         "load_system": 1.7873669496591713,
>         "load_nice": 0,
>         "load_idle": 94.50116688656539,
>         "load_irq": 0,
>         "raw_load": 6309900,
>         "raw_load_user": 4258900,
>         "raw_load_system": 2051000,
>         "raw_load_nice": 0,
>         "raw_load_idle": 108439900,
>         "raw_load_irq": 0
>       },
>       {
>         "load": 7.14431589366737,
>         "load_user": 5.196529737349123,
>         "load_system": 1.9477861563182473,
>         "load_nice": 0,
>         "load_idle": 92.85568410633263,
>         "load_irq": 0,
>         "raw_load": 8220900,
>         "raw_load_user": 5979600,
>         "raw_load_system": 2241300,
>         "raw_load_nice": 0,
>         "raw_load_idle": 106848200,
>         "raw_load_irq": 0
>       },
>       {
>         "load": 7.0672477936043006,
>         "load_user": 5.18023716308505,
>         "load_system": 1.8869236875303756,
>         "load_nice": 0.0000869429888739057,
>         "load_idle": 92.93275220639569,
>         "load_irq": 0,
>         "raw_load": 8128600,
>         "raw_load_user": 5958200,
>         "raw_load_system": 2170300,
>         "raw_load_nice": 100,
>         "raw_load_idle": 106889300,
>         "raw_load_irq": 0
>       },
>       {
>         "load": 5.657756397066548,
>         "load_user": 3.8586858849973567,
>         "load_system": 1.7990705120691914,
>         "load_nice": 0,
>         "load_idle": 94.34224360293345,
>         "load_irq": 0,
>         "raw_load": 6516700,
>         "raw_load_user": 4444500,
>         "raw_load_system": 2072200,
>         "raw_load_nice": 0,
>         "raw_load_idle": 108665000,
>         "raw_load_irq": 0
>       }
>     ]
>   },
>   "memory": {
>     "total": 451776512,
>     "free": 102035456,
>     "used": 349741056,
>     "active": 125931520,
>     "available": 228311040,
>     "buffcache": 223809536,
>     "swaptotal": 104853504,
>     "swapused": 9961472,
>     "swapfree": 94892032
>   },
>   "network": [
>     {
>       "iface": "lo",
>       "ifaceName": "lo",
>       "ip4": "127.0.0.1",
>       "ip6": "::1",
>       "mac": "",
>       "internal": true,
>       "virtual": false,
>       "operstate": "unknown",
>       "type": "virtual",
>       "duplex": "",
>       "mtu": 65536,
>       "speed": -1,
>       "carrierChanges": 0
>     },
>     {
>       "iface": "wlan0",
>       "ifaceName": "wlan0",
>       "ip4": "10.1.10.154",
>       "ip6": "2603:3001:17a:6000:ba27:ebff:fe8e:c48e",
>       "mac": "b8:27:eb:8e:c4:8e",
>       "internal": false,
>       "virtual": false,
>       "operstate": "up",
>       "type": "wireless",
>       "duplex": "",
>       "mtu": 1500,
>       "speed": -1,
>       "carrierChanges": 2
>     },
>     {
>       "iface": "ap0",
>       "ifaceName": "ap0",
>       "ip4": "192.168.10.1",
>       "ip6": "fe80::ba27:ebff:fe8e:c48e",
>       "mac": "b8:27:eb:8e:c4:8e",
>       "internal": false,
>       "virtual": false,
>       "operstate": "up",
>       "type": "wired",
>       "duplex": "",
>       "mtu": 1500,
>       "speed": -1,
>       "carrierChanges": 2
>     }
>   ],
>   "disk": [
>     {
>       "fs": "/dev/root",
>       "type": "ext4",
>       "size": 15385776128,
>       "used": 8967753728,
>       "use": 58.29,
>       "mount": "/"
>     },
>     {
>       "fs": "/dev/mmcblk0p1",
>       "type": "vfat",
>       "size": 264290304,
>       "used": 42488832,
>       "use": 16.08,
>       "mount": "/boot"
>     }
>   ]
> }
> ```
>
> 

`[GET]` **/api/device/macaddress**

Reveals MAC address of device.

> Namespace:  **device**
>
> Response: 
>
> ```json 
> {"macaddress": "b3:27:eb:8e:c4:5e"}
> ```
>
> 

`[GET]` **/api/daemon/start**

Starts the chain daemon on the device.

> Namespace:  **daemon**
>
> Response: 
>
> ```json 
> {"output": "-- daemon response here --"}
> ```
>
> 

`[GET]` **/api/daemon/stop**

Stops the chain daemon on the device.

> Namespace:  **daemon**
>
> Response: 
>
> ```json 
> {"output": "-- daemon response here --"}
> ```
>
> 

`[GET]` **/api/daemon/debug**

Retreives last few lines of debug.log file.

> Namespace:  **daemon**
>
> Response: 
>
> ```json 
> {"debug": ["line1", "line2"]}
> ```
>
> 

`[GET]` **/api/daemon/isrunning**

Indicates whether daemon is running or not.

> Namespace:  **daemon**
>
> Response (Example): 
>
> ```json 
> {"isrunning": true}
> ```

`[GET]` **/api/coin/generatemnemonic**

Generates a 12 word mnemonic phrase

> Namespace:  **coin**
>
> Response (Example): 
>
> ```json 
> {
>   "phrase": "measure price firm picture tattoo inmate hospital concert donate rifle fitness card"
> }
> ```
>
> 

`[GET]` **/api/coin/generateaddressesfrommnemonic**

Generates a 12 word mnemonic phrase

> Namespace:  **coin**
>
> Parameters: 
>
> * **phrase** - 12 word mnemonic phrase
> * **count** - number of private keys to return
> * **type** -  type of address to generate (default: P2PKH)
> * **bip** - bip for address (default: 44)
>
> Response (Example): 
>
> ```json 
> [
>   {
>     "address": "1Crz1U9H9o2dTanwsdJ35tiamWuy6j2MxG",
>     "privateKey": "L2rJUoVKQN3GNyPw8Hmza4XxwsttTeCyABbd7PL7zKzwAXVMRFLf"
>   },
>   {
>     "address": "1GN7KQf7x7LYGeB1TKBAfueDYdmmdbeHQb",
>     "privateKey": "L4hXdqny9AY6AHGTG7VWVZHVgDqN4xYCtE1ZsCAJYdCkdQkrPBUC"
>   },
>   {
>     "address": "12YkkGBdL7Nik8i2k1RT3NnGFUhDdVmDTj",
>     "privateKey": "Kzyb53pJsGS5ggVQwAXYes9WACPWZzUSwCcszatjM9GsoJW8VbM6"
>   },
>   {
>     "address": "1EWLqhswzsPRE3xTTvUc6Thud7BtcN4ykC",
>     "privateKey": "Kwej5zUvbBuNsZSuPG5muDJtrjgnRsEGz6PeQYh4CvR4jugVZRuc"
>   },
>   {
>     "address": "1MoodgLz6xPuaMhqBHKQaycA27YEynVemc",
>     "privateKey": "Ky54HYvq5JrbrLYkMbdCFSwA1EyYFzN64EXMq8WKypnvtgjjGnoP"
>   }
> ]
> ```
>
> 
