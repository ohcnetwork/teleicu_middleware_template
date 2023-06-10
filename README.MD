# TeleICU Middleware Deployment

The documentation is for Equipment Integrators, explaining how to deploy the TeleICU middleware in their development
environment.

## Prerequisites

- You should own a domain to host the TeleICU middleware
- You should have a linux machine to deploy the TeleICU middleware
- Download this [repo](https://github.com/coronasafe/teleicu_middleware_template). The commands in this documentation
  are to be run from inside the downloaded folder.

## Overview

Setting up a test environment for an equipment integrator involves the following processes

- Create facility, equipment asset, and patient in the staging environment of CARE (https://care.coronasafe.in)
- Deploy a hosting solution for the middleware in the local/testing env for CARE staging to be able to reach it.
- Set up TeleICU middleware in local/testing env of the equipment integrator.

## Steps

### 1. Create Facility in CARE Staging

- Login to https://care.coronasafe.in with the credentials below

``` 
username: devdistrictadmin
password: Coronasafe@123 
```

- Go to https://care.coronasafe.in/facility/create and create a facility in **Ernakulam** Disrict. All other values can
  be random.
- Note down the Facility ID from the URL. If the URL is
  https://care.coronasafe.in/facility/1e4fc27a-10dc-42a8-967e-e784c025a96e, then the ID will be
  1e4fc27a-10dc-42a8-967e-e784c025a96e. We will refer to this ID as `FACILITY_ID` in the rest of the documentation.

```
FACILITY_ID=1e4fc27a-10dc-42a8-967e-e784c025a96e
```

- On the facility details page, click on Manage Location, and Add New Location. Give the name as ICU and click Add
  Location.

```
Location=ICU
```

- Click on Manage Beds of the New Location that we created and Add a Bed. As an example, name would be Bed1

```
Bed=Bed1
```

- Go back to the Facility details page and Click on Create Asset. Select the location we just created, and select your
  equipment from the Asset Class. Fill out the mandatory fields and click Create Asset.  
  Example:

```
Asset Name=Monitor 1
Location=ICU
Asset Type=Internal
Asset Class=HL7 Vitals Monitor
Working Status=Working
Customer Support=9876543210 
```

- Note down the Asset ID from the URL. If the URL is  
  https://care.coronasafe.in/facility/1e4fc27a-10dc-42a8-967e-e784c025a96e/assets/03b9f8f5-e2d4-480e-85a0-140c4eb1b837,  
  then the Asset ID will be 03b9f8f5-e2d4-480e-85a0-140c4eb1b837. We will refer to this ID as `ASSET_ID` in the rest of
  the documentation.

```
ASSET_ID=03b9f8f5-e2d4-480e-85a0-140c4eb1b837
```

- From Facility details page, click on Add Patient Details.
- Fill out the mandatory fields to create a patient. Do the same for the Consultation page that comes after it.
- On the Patient Dashboard Page, click on Assign Bed near to the Patient Picture on top of the Page.
- Assign the Bed we created before, and click on Move to Bed.

## 2. Hosting solution in local/testing env

### Pre-requisite: You should own a domain to provide hostname to the middleware.

#### If you already have a hosting solution

For CARE Staging to access the TeleICU middleware, the middleware should have a hostname. If you already have a static
public IP and a hosting solution like nginx already present in the local/testing env, create a subdomain that will
point to the public IP and to port 8001.

#### If you don't have a static public IP

In this scenario, we could use a tunneling solution to reach middleware in a system that does not have a static public
IP.  
We will use cloudflare tunnel for this purpose.

- Create a cloudflare account and move your domain to cloudflare.
- Install Cloudflare cli in local. Run the following script on your terminal to install cloudflared.

```
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared '"$(lsb_release -cs)"' main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt-get update && sudo apt-get install cloudflared -y
```

- Authenticate using the below command.

```
cloudflared tunnel login
```

- Create a tunnel using the following command.

```
cloudflared tunnel create middleware-tunnel
```

- A tunnel with the name `middleware-tunnel` will be created. Note down the UUID of the tunnel from the output screen.
- Use this UUID to replace `tunnel-id` in `config.yml` file. And update the hostname as your preferred hostname.
  Example:

```
tunnel: a092e1bf-9eab-45f0-9cee-7928e1054349
credentials-file: /home/teleicu/.cloudflared/a092e1bf-9eab-45f0-9cee-7928e1054349.json

ingress:
  - hostname: middleware.example.com
    service: http://localhost:8001
  - service: http_status:404
```

- Use the same UUID to create a CNAME record in cloudflare DNS with your preferred hostname.  
  Example:

```
Type: CNAME
Name: middleware
Target: a092e1bf-9eab-45f0-9cee-7928e1054349.cfargotunnel.com
```

- Copy the `config.yml` to `$HOME/.cloudflared` directory

```
cp config.yml "$HOME"/.cloudflared
```

## Deploy TeleICU Middleware

- Update the `.env` file. Replace values for hostname, and `FACILITY_ID`
  Example:

```
HOSTNAME=middleware.example.com
USERNAME=username
PASSWORD=password
PORT=8090

FACILITY_ID=2ad6f9d4-c3d4-4a2c-a98d-08ae9bbe4924
CARE_API=https://careapi.coronasafe.in

POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=teleicu_middleware
DATABASE_URL=postgres://postgres:postgres@middleware_db:5432/teleicu_middleware?schema=public

SENTRY_DSN=https://d7b96cc4918e91997e61f0bc20f64067@177311o.ingest.sentry.io/6584654
SENTRY_ENV=middleware_example_com
```

- Install Docker

```
sudo curl -fsSL get.docker.com | sh
sudo usermod -aG docker "$USER"

sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

- Run TeleICU middleware using docker

```
docker compose up -d
```

## Integrating all together

- Install cloudflared as a service

```
sudo cloudflared service install
sudo systemctl enable cloudflared
```

- Update the asset in CARE Staging
    - Visit the asset details page
    - Click on configure and update the hostname, IP address of the device
    - Assign Bed as the Bed we created earlier.
- Update the asset in TeleICU Middleware
    - Visit the asset details page at `localhost:8090/assets`
    - Add Name, Description, `ASSET_ID`, IP address of the device.

## Send Data from Equipment to Middleware

### Request Details

#### Request Endpoint

- http://localhost:8001/update_observations

#### Request Method

- POST

#### Data Format

```
[
  [
    observation(json),
    observation(json)
  ]
]
```

A single observation is represented as a JSON string, based
on [FHIR observation resource](https://build.fhir.org/observation.html), with observed parameter name as key and
observed parameter value as value.
Example:

```
    {
      "observation_id": "heart-rate",
      "device_id": "192.168.1.13",
      "date-time": "2022-06-03 08:21:48",
      "patient-id": "3",
      "patient-name": "PATIENT 3",
      "status": "Message-Leads Off",
      "value": null,
      "unit": "bpm",
      "interpretation": "NA",
      "low-limit": 40,
      "high-limit": 120
    }
```

The actual payload will be a two-dimensional array of these objects.

An example of the payload is given below

```
[
  [
    {
      "observation_id": "heart-rate",
      "device_id": "192.168.1.13",
      "date-time": "2023-02-20 18:00:43",
      "patient-id": "3",
      "patient-name": "PATIENT 3",
      "status": "Message-Leads Off",
      "value": null,
      "unit": "bpm",
      "interpretation": "NA",
      "low-limit": 40,
      "high-limit": 120
    },
    {
      "observation_id": "ST",
      "device_id": "192.168.1.13",
      "date-time": "2023-02-20 18:00:43",
      "patient-id": "3",
      "patient-name": "PATIENT 3",
      "status": "Message-Leads Off",
      "value": null,
      "unit": "mV",
      "interpretation": "NA",
      "low-limit": -0.3,
      "high-limit": 0.3
    },
    {
      "observation_id": "SpO2",
      "device_id": "192.168.1.13",
      "date-time": "2023-02-20 18:00:43",
      "patient-id": "3",
      "patient-name": "PATIENT 3",
      "status": "final",
      "value": 95,
      "unit": "%",
      "interpretation": "normal",
      "low-limit": 90,
      "high-limit": 100
    },
    {
      "observation_id": "pulse-rate",
      "device_id": "192.168.1.13",
      "date-time": "2023-02-20 18:00:43",
      "patient-id": "3",
      "patient-name": "PATIENT 3",
      "status": "final",
      "value": 71,
      "unit": "bpm",
      "interpretation": "normal",
      "low-limit": 40,
      "high-limit": 120
    },
    {
      "observation_id": "respiratory-rate",
      "device_id": "192.168.1.13",
      "date-time": "2023-02-20 18:00:43",
      "patient-id": "3",
      "patient-name": "PATIENT 3",
      "status": "Message-Leads Off",
      "value": null,
      "unit": "brpm",
      "interpretation": "NA",
      "low-limit": 6,
      "high-limit": 40
    },
    {
      "observation_id": "body-temperature1",
      "device_id": "192.168.1.13",
      "date-time": "2023-02-20 18:00:43",
      "patient-id": "3",
      "patient-name": "PATIENT 3",
      "status": "final",
      "value": 76.5,
      "unit": "deg F",
      "interpretation": "low",
      "low-limit": 86.0,
      "high-limit": 107.6
    },
    {
      "observation_id": "body-temperature2",
      "device_id": "192.168.1.13",
      "date-time": "2023-02-20 18:00:43",
      "patient-id": "3",
      "patient-name": "PATIENT 3",
      "status": "Message-Measurement Invalid",
      "value": null,
      "unit": "deg F",
      "interpretation": "NA",
      "low-limit": 86.0,
      "high-limit": 107.6
    },
    {
      "observation_id":"waveform",
      "device_id":"192.168.1.13",
      "date-time":"2023-02-20 18:00:43",
      "patient-id":"3",
      "patient-name":"Patient 3",
      "wave-name":"II",
      "resolution":"1.5uV",
      "sampling rate":"250/sec",
      "data-baseline":2047,
      "data-low-limit":0,
      "data-high-limit":4095,
      "data":"2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047 2047"
    },
    {
      "observation_id":"waveform",
      "device_id":"192.168.1.13",
      "date-time":"2023-02-20 18:00:43",
      "patient-id":"3",
      "patient-name":"Patient 3",
      "wave-name":"Pleth",
      "resolution":"NA",
      "sampling rate":"100/sec",
      "data-baseline":0,
      "data-low-limit":0,
      "data-high-limit":255,
      "data":"87 87 87 86 85 82 80 78 76 74 71 70 68 66 65 64 62 62 61 60 59 57 56 55 53 52 51 50 49 47 46 45 43 42 40 38 37 36 35 35 35 36 37 42 45 49 57 61 66 73 76 80 82 84 87 88 89 90 90 89 88 87 85 83 81 77 75 73 70 69 67 65 64 63 63 62 60 59 57 56 55 53 51 51 50 49 47 45 44 42 40 39 38"
    },
    {
      "observation_id":"waveform",
      "device_id":"192.168.1.13",
      "date-time":"2023-02-20 18:00:43",
      "patient-id":"3",
      "patient-name":"Patient 3",
      "wave-name":"Respiration",
      "resolution":"NA",
      "sampling rate":"100/sec",
      "data-baseline":0,
      "data-low-limit":0,
      "data-high-limit":255,
      "data":"127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127 127"
    }
  ]
]
```

### Post Deployment

- Visit the Patient Dashboard page to see the data coming from the Asset that is linked to the bed on which patient is
  assigned/moved.

## Mock Data Requests

The docker file also contains a mock data generator that can be used to generate vitals data for testing purposes.
Uncomment the following lines in the docker-compose.yml file to enable the mock data generator.

```
#  mock_cns_req:
#    restart: always
#    image: ghcr.io/coronasafe/mock-cns-req:latest
#    container_name: mock_cns_req
#    ports:
#    - "8002:8002"
#    logging:
#      options:
#        max-size: "10m"
#        max-file: "3"
```

The generator send continuous mock data to the middleware for 10 devices with IP addresses ranging from
192.168.1.11 to 192.168.1.20.

## Connect a Camera Feed

The middleware setup also consists of an [RTSPtoWeb](https://github.com/deepch/RTSPtoWeb) player that stream a feed 
from the camera. The player can be accessed at http://localhost:8080. In the player, add a stream and 
give the RTSP URL of the camera feed. The player will then stream the feed from the camera. 
> RTSP URL can vary from device to device but a general look is as below:  
> ```
> rtsp://<username>:<password>@<camera IP>:<port>/stream
> ```

Similar to adding a Vitals Monitor, add a new asset with asset type as `ONVIF Camera` and give the following details:
```
IP Address:
username:
password:
Camera ID:
```
The `Camera ID` can be found in the RTSPtoWeb player URL when you open the camera feed.