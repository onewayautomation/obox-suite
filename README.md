# Introduction

New product ``oBox Suite`` offered by One-Way Automation includes oBox Industrial PC and suite of multiple applications. These applications are intended to connect to industrial data sources such as PLCs, harmonize and transform this data, and then forward it to various databases or MQTT brokers.

Main applications distributed by One-Way Automation are: 

- ``DeviceGateway`` to make data from different industrial sources accessible over OPC UA Server interface. If your data source already provides data via OPC UA interface, you can skip using of this application.

- ``OPC UA Forge``. Use it if you need to model your namespace, harmonize and normalize data, and publish data to MQTT brokers using OPC UA Pub/Sub payload format. As well as it provides access to the data over REST API, and can act as a Historian.

- ``ogamma Visual Logger for OPC`` - this application is used to move data from OPC UA servers to various databases, such as SQL servers, InfluxdB, Apache Kafka, or MQTT brokers.

Depending on your use case, you can use each of them separately, or together in any combination.

Additionally, you can run other applications in the oBox Industrial PC, such as open-source products Grafana, InfluxDB, TimescaleDB, Grafana, or commercial products like Ignition.

This folder has docker compose and other accompanying configuration files used to create and run all applications from the ``oBox Suite`` as Docker containers. While mainly they are intended to run in the ``oBox Industrial PC``, you can run them in a regular Docker engine such as Docker Desktop for Windows.

Before starting containers, you will need to perform some configuration steps as described further below. For quick evaluation tests and proof of concept projects, you can skip configuration steps related to security and user authentication and authorization, as described in the Quick Start section below.

Note: This repository is intended to use for demo and learning purposes. It can be taken as a base for production configuration, but container image versions and security related settings must be revised before using in production.

# Quick Start with default settings.

By default, containers are configured with base domain name ``test-co.opcfy.io``. You can leave it as is, and configure your system to resolve this name to the IP address of the Docker host machine. In Windows machines this can be done by modifying file ``C:\Windows\System32\drivers\etc\hosts``. Example entries can be found in the file ``hosts.txt`` in this repository.

The minimal step to start conatiners with default settings is:

1. Open command line terminal and navigate to the folder ``oBox`` of the repository.

	First, in order to pull container image for OPC UA Forge, login to its registry by running the command:

	```
	docker login -u UserName -p Password ogamma.azurecr.io
	```

	Hint: contact support@onewayautomation.com to get ``UserName`` and ``Password`` credentials.

	After succesful login, start containers:

	``` 
		docker compose up -d
	```

2. After starting of containers, main applications should be accessible using their direct endpoints.

	Application - specific  endpoints which can be accessed directly bypassing reverse proxy listed below. If accessing from the Docker host machine, you can use ``localhost`` as a host name.

	- http://localhost:4880 - ogamma Visual Logger for OPC. Default user name and password: admin/password.
	- http://localhost:3280 - DeviceGateway. Default credentials: administrator/admin. OPC UA Server port is 52220
	- http://localhost:8990 - OPC UA Forge. Will ask for license file and create user credentials in the first connection. Contact support@onewayautomation.com for the license file.


# Provisioning for production environment.

oBox Suite in production environment is expected to run behind reverse proxy, and the GUI should be accessed over secured https connection with valid TLS certificates. Also, the proxy web server requires user authentication and authorization using identity provider. All this requires some configuration steps as described below.

## Adjust host names and credentials.

If you plan to access ``oBox Suite`` from local network only, you are free to assign host names as you wish. Resolution of host names to IP addresses should be configured using local network DNS settings or by modifying of the system file ``C:\Windows\System32\drivers\etc\hosts``.

To set custom host names, edit environment variables in the file ``.env``. 

Note: Don't forget also change credentials from default values in the file ``.env``. Do not use default values in production!

Note: in case if you plan to access oBox configuration GUI remotely from the Internet, you might need to register primary domain name with a domain registrar, and use host names as sub-domains of this primary domain. Host names for our test company ``test-co.opcfy.io`` is composed following this approach.

## Update docker-compose.yml and container configuration files with settings from ``.env`` file.

Navigate to the folder ``oBox`` and run the command:

``` 
	python.exe ../UpdateConfiguration.py
```
As a result, the file ``docker-compose.yml`` will be generated from the template file, with environment variables replaced by values from the file ``.env``.
Also, this script generates file ``grafana/grafana.ini``, with changes to run behind the reverse proxy. Also file ``nginx/Dockerfile`` will be updated. 

## TLS certificates.

All applications are configured using web based interface. All of them except identity provider ``keycloak`` recommended to be accessed via reverse proxy ``nginx``, over secure https protocol. This approach simplifies certificates management, reducing number of required TLS certificates to 2: one for nginx service and one for keycloak service.

Note: steps below can look little bit complicated, later these steps will be automated.

## Generate TLS certificate for nginx reverse proxy server.

### Required input data.

- Host name of the reverse proxy container - value of the variable ``REVERSE_PROXY_HOST`` defined in file ``.env``. When this address is browsed by Internet browser, it shows main configuration page of the oBox Suite. For purposes of this guide lets set it to the value ``obox.test-co.opcfy.io``.
-  Other fields such as common name, country, locality, organization are generic fields used in x509 TLS certificates.
  
### Create the TLS certificate signed by OPC UA Certificate Authority.  

- Open OPC UA Registration Authority web page https://public.opcua.ca/ejbca/ra/  Note: The URL must have trailing forward slash symbol at the end.

	Note: this certificate authority is used for test purposes only. For production, use your own PKI infrastructure to manage certificates. If you need help on setting up your own PKI, please contact support@onewayautomation.com.

	Note: when you open this URL, it will redirect you to the cloudflare login page and ask to enter email address and then to enter a code sent to this email. In this case, first send email to support@onewayautomation.com requesting to authorize your email address to use the service. Continue to the next steps after getting approval response from One-Way Automation support.

- Click on the button ``Make New Request``
- In the field ``Certificate Type``, select option ``TLS Server Profile``
- If you have Certificate Sign Request (CSR) already generated, use option ``Provided by user``. Otherwise, select option ``By the CA``. Further below it is assumed that the later option is selected.
- Fill fields in the form. Make sure that the field DNS name is set to the same values as ``REVERSE_PROXY_HOST`` defined in file ``.env``.
- Click on button ``Download PEM``.
- Save the file in folder ``sharedFolder/nginx``.
- Open saved file in text editor.
- Copy part of the text starting from ``-----BEGIN PRIVATE KEY-----`` to the ``-----END PRIVATE KEY-----`` inclusive into clipboard.
- Open file ``nginx/own-cert/nginx-key.pem`` and replace the content with the value from clipboard, and save file.
- Go back to the file with generated certificate and copy all 3 certificates (text starting by the first ``-----BEGIN CERTIFICATE-----`` and ending by the last ``-----END CERTIFICATE-----``, inclusive).
- Open file ``nginx/own-cert/nginx-cert.pem``, replace the content with clipboard value, and save the file.
- Go back to the file with generated certificate and copy the last certificate (text starting by the last ``-----BEGIN CERTIFICATE-----`` and ending by the last ``-----END CERTIFICATE-----``, inclusive).
- Open file ``nginx/ca-certs/ca_certs.crt`` and replace content with the clipboard text, and save the file.

## Generate TLS certificate for Identity Provider service (keycloak).

The steps are similar to generating of the certificate for nginx service.

- Open OPC UA Registration Authority web page https://public.opcua.ca/ejbca/ra/

	Note: this certificate authority is used for test purposes only. For production, use your own PKI infrastructure to manage certificates. If you need help on setting up your own PKI, please contact support@onewayautomation.com.

	Note: it might ask to enter email address and then to enter a code sent to this email. In this case, first send email to support@onewayautomation.com requesting to authorize your email address to use the service. Continue to the next steps after getting approval response from One-Way Automation support.

- Click on the button ``Make New Request``
- In the field ``Certificate Type``, select option ``TLS Server Profile``
- If you have Certificate Sign Request (CSR) already generated, use option ``Provided by user``. Otherwise, select option ``By the CA``. Further below it is assumed that the later option is selected.
- Fill fields in the form. Make sure that the field ``DNS Name`` is set to the same values as ``KEYCLOAK_HOST`` defined in the file ``.env``.
- Click on button ``Download PEM``.
- Save the file in folder ``sharedFolder/keycloak``.
- Open saved file in text editor.
- Copy part of the text starting from ``-----BEGIN PRIVATE KEY-----`` to the ``-----END PRIVATE KEY-----`` inclusive into clipboard.
- Open file ``keycloak/certs/keycloak.pem`` and replace the content with the value from clipboard, and save file.
- Go back to the file with generated certificate and copy all 3 certificates (text starting by the first ``-----BEGIN CERTIFICATE-----`` and ending by the last ``-----END CERTIFICATE-----``, inclusive).
- Open file ``keycloak/certs/keycloak.crt``, replace the content with clipboard value, and save the file.
- Go back to the file with generated certificate and copy the last certificate (text starting by the last ``-----BEGIN CERTIFICATE-----`` and ending by the last ``-----END CERTIFICATE-----``, inclusive).
- Open file ``keycloak/certs/ca_certs.crt`` and replace content with the clipboard text, and save the file.

## Start and configure containers

Open command line terminal and navigate to the folder ``oBox`` of the repository.

First, in order to pull container image for OPC UA Forge, login to its registry by running the command:

```
docker login -u UserName -p Password ogamma.azurecr.io
```

Hint: contact support@onewayautomation.com to get ``UserName`` and ``Password`` credentials.

After succesful login, start containers:

``` 
	docker compose up -d
```

After starting containers, if this is the very first run, databases will be created and default configuration files will be created.

## Application endpoints

Before accessing all application through single reverse proxy endpoint, first, identity provider service ``keycloak`` needs to be configured, and then authentication proxy configuration needs to be updated. In the local network, you can bypass these settings and access each application configuration web GUI independently, using their own endpoints. In the examples below default host name ``test-co.opcfy.io`` is used:

- https://keycloak-test-co.opcfy.io:8443 - ``keycloak`` identity provider service. Default credentials are admin/password. Need to add client application as described below. 
- https://test-co.opcfy.io - proxy server endpoint. Will not work right away, to make it work, service ``oauth2Proxy`` needs to be configured with authentication token from keycloak service (environment variables ``OAUTH2_PROXY_CLIENT_SECRET`` and ``OAUTH2_PROXY_CLIENT_ID``).

Application - specific  endpoints which can be accessed directly bypassing reverse proxy. If accessing from the Docker host machine, you can use ``localhost`` as a host name.

- http://localhost:4880 - ogamma Visual Logger for OPC. Default user name and password: admin/password.
- http://localhost:3280 - DeviceGateway. Default credentials: administrator/admin. OPC UA Server port is 52220
- http://localhost:8990 - OPC UA Forge. Will ask for license file and create user credentials in the first connection.

- http://localhost:8086 - InfluxDB 2. Will ask to create user credentials in the first connection.
- http://localhost:8084 - InfluxDB 1.8. There is not GUI at this port, for API connections only. 
- http://localhost:8085 - GUI to access InfluxDB 1.8
- http://localhost:3000 - Grafana. Default passord is ```admin``.

## Configuring ``keycloack`` and ``oauth2proxy`` services.

Once the ``keycloak`` service starts, it needs some configuration steps.

Login at the keycloak web page (can be accessed on port 8443). Login credentials can be found in file ``.env`` - by default it is admin/password. 

- Click on the ``Clients`` link in the left side panel. As a result, the ``Clients`` panel will be opened in the the main screen area.
- Click on the button ``Create client``
    
    The Create Client wizard starts with 3 pages. Change the following below fields.

 
    * ``General settings``.

        * ``Client ID``: arbitrary identifier. For example, ``opcfy``. It will be required to configure the service ``oauth2proxy``.

		 
    * ``Capability config``.
	 
        * ``Client Authentication`` - turn ON
        * ``Authorization`` - turn ON.

    * ``Login settings``

        - ``Valid redirect URIs``: enter ``*``
        - ``Web origins`` - enter ``*``.


    Click on the ``Save`` button to complete creating of the new client record. Select the record and edit some more fields.

- In the ``Settings`` tab, fill the field ``Admin URL``, for example, ``https://keycloak.test-co.opcfy.io:8443/admin``.

- In the ``Credentials`` tab

    - set option ``Client Authenticator`` to ``Client ID and Secret``. 
    - copy value of the field ``Client Secret``. If will required to congure the service ``oautg2proxy``.

- ``Advanced`` tab:

  * ``Access token signature algorithm`` field: set to ``RS256``.

  * ``Advanced settings / Access token lifespan``: set to 15 minutes.

- In the file ``docker-compose.yml``configure ``oauth2proxy`` container settings so it could connect to the keycloak service:
  
	- Copy value of the field ``Client Secret``. In the ``oauth2proxy`` service, set environment variable ``OAUTH2_PROXY_CLIENT_SECRET`` to this value. 
	- Also, set values of the environment variable ``OAUTH2_PROXY_CLIENT_ID``to the ``Client ID`` field from ``Settings`` tab.
	- Restart container to apply settings.
		
- Configure access roles.

  Roles assigned to the logged in users should be configured as described in the User Manual: https://onewayautomation.com/visual-logger-docs/html/configure.html#versions-4-0-0-or-newer

	Essentially, to create roles and assign them to the user:
	
	- To create roles, on the left side panel select ``Realm roles``, click on ``Create role`` button, and create roles ``ovl-admin``, ``ovl-read``, ``ovl-write``.
	- To assign roles to the specific user, select ``Users`` in the left side panel, select desired user (by default there is one user ``admin``). Click on the user record, select tab ``Role mapping``, click ``Assign role`` button, select roles to assign, and click on the ``Assign`` button.
	- Also assign roles for the client application: in the left side panel select ``Clients``, select the client (by default ``opcfy``), open ``Roles`` tab, and add roles as required (``ovl-read`` and optionally ``ovl-write`` or ``ovl-admin``).

- Configure users: enter email address field (version 4.0.0 has issue if user record has empty ``email`` field).
  
## Configuring ``tunnel`` service.

The service ``tunnel`` (used to access the box remotely from the Internet using cloudflare services) needs to be configured: environment variable ``TUNNEL_TOKEN`` should be set to the value of the token from cloudflare Zero Trust tunnel settings. For details contact support@onewayautomation.com.

## Configure DNS settings or hosts file.

Make sure that host name for the reverse proxy and the ``keycloak`` service can be resolved to correct IP addresses. This can be done by DNS settings, or by modifying file ``C:\Windows\System32\drivers\etc\hosts`` (in Windows machines).

## Trust to OPC UA CA certificates.

If browser warns that certificates are not trusted, you can download OPC UA Root CA certificate and OPC UA Issuing CA Certificate from this link: ``https://public.opcua.ca/ejbca/ra/cas.xhtml``. And then import the root certificate into the trusted root certificates location of the OS, the second certificate - into trusted intermediate certificates location.

## Licensing of oBox applications.

The ``oBox Suite`` can include bunch of open source applications such as PostgreSQL or InfluxDB or Grafana or others, which do not require activation by a license key to start using them. Meanwhile, 3 applications require activation using licenses. For evaluation, you can use free licenses, but they should be obtained from One-Way Automation and applied:

- ``DeviceGateway``, from our partner Takebishi: for free evaluation licenses, as well as for production licenses, please contact sales@onewayautomation.com, or send a message using online form https://onewayautomation.com/contact-us . 
- ``ogamma Visual Logegr for OPC``: you can get a free evaluation license for it at our online store: https://onewayautomation.com/online-store#. Activation of this application is described in online User Manual: https://onewayautomation.com/visual-logger-docs/html/activate.html#activate
- ``ProsysOPC UA Forge``, from our partner ProsysOPC: 

	- for free evaluation licenses, as well as for production licenses, please contact sales@onewayautomation.com, or send a message using online form https://onewayautomation.com/contact-us .
	- Container image for this product is hosted in private repository. For access credentials contact sales@onewayautomation.com

# Troubleshooting

Sometimes when Docker for Windows is used, you might get error that a container cannot be started because the TCP port cannot be opened due to permission isues.
In this case, start Windows command line terminal with Administrator rights, and run the command:

```
netsh int ip show excludedportrange tcp
```

If the port number exposed by the container falls into one of the ranges, then it means that this range is reserved by the operating system and cannot be used.

You can try to delete specific range by running command like:

```
netsh int ip delete excludedportrange tcp 52196 100
```

The last two numbers are starting port number and number of records. In this example this will result to the range from 52196 to 52295.

If this command fails with permission denied error, the most possible reason is because this range is within range reserved for dynamically assigned ports. You can change range of dynamically assigned ports with command like:

```
netsh int ipv4 set dynamic tcp start=55000 num=10000
```
For this command to take effect you need to restart the computer.

In this example, dynamically assigned ports range is moved to the higher range, starting from 55000, so the port number we are interested (52220) is out of that range and becames free to use.
