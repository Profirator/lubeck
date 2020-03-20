## placeholder for master documentation

DRAFT DRAFT DRAFT DRAFT DRAFT DRAFT DRAFT DRAFT DRAFT DRAFT DRAFT DRAFT

### Preface
This basic is documentation of FIWARE based PoC setup for the city of Lübeck. Basic documentation is:
Overview of structure, functionality and basic usage.

Basic documentation is not:
- Complete tutorial on every feature of the FIWARE components.
- Debugging reference.

The FIWARE PoC for the city of Lübeck is:
- Proof of Concept on how the FIWARE and other open source components work together.

The FIWARE PoC for the city of Lübeck is not:
- Production ready
- High available


### Overall Architecture / structure

The following components are deployed:

- Orion Context broker for real time data
- Quantum leap for historical data
- Apache Nifi
- Identity manager
- Grafana for data visualizations
- API management
- API proxy
- Basic map Visualization
- Open Data Portal (CKAN)
- Wirecloud Portal

High level architecture:
![Architecture](images/Archtecture2.png)

More information on the components can be found on [FIWARE catalog page](https://www.fiware.org/developers/catalogue/), and on their respective sites (Apache Nifi and Grafana). Basic map visualisation is developed from scratch for this project.

### UML Diagram
TBD

### Onboarding users / new user creation

Sign up at https://accounts.lubeck.apinf.cloud/ (you need to confirm email address)

Sign in at https://apis.lubeck.apinf.cloud/ using FIWARE login

The NGSI V2 API is exposed at https://context.lubeck.apinf.cloud/v2/ and

https://sthdata.lubeck.apinf.cloud/ql/ for historical data

Accesses for data can be handled by using Oauth Bearer tokens. Once you have signed in, you can fetch a token from platform:

![images/oauthtoken.PNG](images/oauthtoken.PNG)

This will allow you to access all the tenants on the Context Broker you have access to. Please see [documentation](https://apinf-fiware.readthedocs.io/en/latest/#tenant-manager-ui) on how to add tenants.

### Component overview and documentation:

Tenant-manager

How to is covered in: [Tenant-manager documentation](https://apinf-fiware.readthedocs.io/en/latest/#tenant-manager-ui)

Grafana

Access via https://charts.lubeck.apinf.cloud/ the admin access is secured by password, which is in the grafana.yml Otherwise, Grafana usage is standard; connect database:
![grafana1](images/grafan-postgres.PNG)

and configure the charts you need:
![grafana temp](images/grafana-temp.PNG)

example SQL query:
```
SELECT
  dateobserved AS "time",
  cast(temperature as float), entity_id
FROM  mtweatherobserved.etweatherobserved
ORDER BY 1
```

Basic map Visualisation

Landing page: https://gis.lubeck.apinf.cloud/ holds two sub pages, one for static data and another one with Weather observed and ParkingSpot. Source code is in github: https://github.com/Profirator/lubeck

Orion Context broker for real time data and Quantum leap for historical data. Are accessed via NGSI v2 API. Their respective documentation can be found [here](https://fiware-orion.rtfd.io/) and [here](https://github.com/smartsdk/ngsi-timeseries-api/).

source code is under this repo. You can build docker container:
```
docker how to: docker stop leaflet01 ; docker rm leaflet01 ; docker build -t {org}/{tag}:{version} . ; docker run -dit --name leaflet01 -p 8181:8181 {org}/{tag}
```

Apache Nifi
End users shall not access / use Apache Nifi. Configuration is described later in the documentation.

Identity manager

Identity manager (keyrock) is configured and needed for initial user account creation. More documentation [here](https://fiware-idm.readthedocs.io/en/latest/)

API management 
Usage in relevant parts are described in this documentation. More information can be found [here](https://github.com/apinf/platform)

API proxy
End users shall not access API-umbrella. 
Is based on the NREL/Api-umbrella. NREL documentation is [here](https://api-umbrella.readthedocs.io/en/latest/)

Open Data Portal (CKAN) 
Is installed, but not configured. Documentation can be found [here](https://fiware-ckan-extensions.rtfd.io/)

Wirecloud Portal
Is installed, but not configured. Documentation can be found [here](https://wirecloud.rtfd.io/)

Broker subscriptions
To make a subscription so that data from Orion context broker is persisted in Quantum Leap / Crate DB, you need to make (POST to https://context.lubeck.apinf.cloud/v2/subscriptions) a subscription. An example: 
´´´
{
        "description": "nifi11 test sub for mongo/crate interaction",
        "subject": {
          "entities": [{ "idPattern": ".*" }],
          "condition": { "attrs": [] }
        },
        "notification": {
          "attrs": [],
          "http": { "url": "http://quantumleap:8668/v2/notify" },
          "metadata": ["dateCreated", "dateModified", "timestamp"]
        }
      }
´´´
more on subscriptios in Orion Context broker documentation.

### Niota connection

The PoC is getting it's real time data from a Niota platfrom. Information is routed via Apache Nifi, and fed to Orion Context broker. In Niota, there is an mqtt consumer, which the Apache Nifi is subscribing to.

The requirement is that the is a consumer provided by niota administrator. If niota is not available, the data is not available on the PoC platfrom.

### Connecting new datasources

This section applies when data is available via mqtt topic, which can be subscribed to. Inorder to access the data, you need to have credentials to subscribe to the Niota provided mqtt broker.

1) Sign in to Niota.
2) Look up for the mqtt credentials in Niota.
3) Check what kind of payload you get from devices.
4) Plan what data do you need and can use from the payload.
5) Check for existing FIWARE datamodel in [FIWARE datamodels](https://www.fiware.org/developers/data-models/). If not, create new one following the guidelines and consider contributing to FIWARE.
6) Design a Apache Nifi flow. Current configuration is given in section "Apache Nifi configuration"

### Apache Nifi configuration
Currently Apache Nifi is collecting data from Niota mqtt server. The data that is arriving is Environmental data (WeatherObserved data model) and Parking data (ParkingSpot data model).

Process flow in high level is as follows:
1) ConsumeMQTT processor listens to Niota mqtt broker and subscribes to a topic tree:consumers/35/apps/+/devices/+/fiware
2) EvaluateJsonPath receives traffic and attaches an "$.application_id" property to the flow file.
3) RouteOnAttribute processor looks at the application id variable and flow is directed to four different processors: one for WeatherObserved and three for different parking sensors:
```
3 Types of Sensors, 3 different payloads for the status
Libellium   
Application	CBB Consulting Parkraum Überwachung
application_id	38
device_type_id	91
occupied	true/false
	
Bosch  
Application	Parking Travemünde
application_id	39
device_type_id	90
parked	0/1

PNI PlacePod	
Application	Parking	
application_id	36	
device_type_id	56	
status	0/1
```
4) JoltTransformJSON transforms the JSON into NGSI datamodel and pushes it into 
5) InvokeHTTP processor, which then POSTs the data to Orion Context Broker.

Here is the UI flow:
![images/nifi1.PNG](images/nifi1.PNG)
and the flow file is backed up in: [flow file](flow.xml.gz)

if you need to restore the flow file, copy it to the nifi container to /opt/nifi/nifi-current/conf

### API Management configuration

Please refer to [configure.md](configure.md)
### Grafana configuration
Please refer to [configure.md](configure.md) and configuration files in the repo.

### Setting up with configuration files
Please refer to [configure.md](configure.md)

### High Availability
System is not configured for High Availability.

### Options for Restoring system state after failure
Backups are not configured. Databases are mapped as volumes to localhost disk. 

### Security

On the server, only ports 443 and 22 are exposed. To access apache Nifi, you need to tunnel to the server and Apache Nifi is available in port 8080.

The access to the services is handed via TLS (only HTTPS connections are allowed). Certificates are Let's Encrypt.

For fetching data from tenant, Oauth2 tokens are used.