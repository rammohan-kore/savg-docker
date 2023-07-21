# docker-compose

This folder contains a docker compose file that can be used to run a jambonz system for development purposes on your laptop or other docker machine.  It is not intended to be used for production purposes.

The docker network will include all of components of a jambonz system:
- a Session Border Controller (SBC)
- a feature server
- mysql database
- redis server
- web provisioning application

## Configuration
There are a few configuration items you will need to supply before running docker-compose:

1. A json key from google cloud to use for text-to-speech and speech-to-text.  Download the key from [GCP](https://console.cloud.google.com/) and save it as gcp.json in the [credentials subfolder](credentials/).

2. Edit the [.env](.env) file and minimally specify `HOST_IP` as the public ip address of the host machine on which you are running docker. 
>>If you are testing by using a softphone on your laptop you can simply use the loopback address; e.g. `HOST_IP=127.0.0.1`, otherwise specify the IP address that external SIP traffic will be sent to.

3. Optionally, if you want to use AWS/Polly for text-to-speech then specify your AWS credentials (access key id, secret access key, and region) in the [.env](.env) file.

## Running
Checkout and install as follows:
```
git clone https://github.com/rammohan-kore/savg-docker.git
cd savg-docker/docker
git submodule update --init

# now copy json key to credentials/gcp.json and edit .env as described above

# now start up the docker network
docker-compose -f docker-compose.yaml up -d
```
This step will take several minutes the first time you start it as several docker images need to be downloaded and built.  Once all of the containers are running, it may take another minute for the webapp to be built and connect to the database.  

Once it does, you can log into the webapp at `localhost:3010` and configure the system.  As usual, the initial password will be admin/admin and you will be forced to change it.

You can then configure your account, applications, sip trunks and phone numbers as usual.

To stop the system, from the same directory simply run:
```
docker-compose -f docker-compose.yaml down
```
To view logs use
docker-compose logs  -f --tail=100
docker-compose logs  -f --tail=100 feature-server sbc-outbound


**Note**: The mysql database will be stored in the `data_volume/` subfolder that is created, so that your provisioning data will persist between restarts of the docker network.

Once you have provisioned the system, you should be able to point sip devices to register and send INVITEs to port 5060 on host ip address that you configured above and have that traffic routed to your jambonz system.

## Capacity
The system has limited capacity as it is intended to be used as a personal development / test machine.  

Specifically, the rtpengine container which is part of the SBC function is configured to handle at most 50 simultaneous calls.

Misc Notes:
------------
Change the ipaddress in sbc/hosts file - This IP address you can obtain using ifconfig command

After all the docker instances are up & running do the following:
login to redis using redis-cli -h 172.10.0.3
flushall
SADD jb:active-fs "172.10.0.50:5070"
SADD jb:fs-service-url "http://172.10.0.60:3012"


Installing another freeswitch instance to use it as a SIP infrastructure for end customers
docker run -it --name freeswitchpbx --net=host  -v /home/pocs:/pocs debian:bullseye
In the /home/pocs folder create a bash script install-freeswitch.sh with content below
#!/bin/sh
TOKEN=pat_cb6LCRmVoq7N5BynA22Yc7gr
apt-get update && apt-get install -y gnupg2 wget lsb-release

wget --http-user=signalwire --http-password=$TOKEN -O /usr/share/keyrings/signalwire-freeswitch-repo.gpg https://freeswitch.signalwire.com/repo/deb/debian-release/signalwire-freeswitch-repo.gpg

echo "machine freeswitch.signalwire.com login signalwire password $TOKEN" > /etc/apt/auth.conf
chmod 600 /etc/apt/auth.conf
echo "deb [signed-by=/usr/share/keyrings/signalwire-freeswitch-repo.gpg] https://freeswitch.signalwire.com/repo/deb/debian-release/ `lsb_release -sc` main" > /etc/apt/sources.list.d/freeswitch.list
echo "deb-src [signed-by=/usr/share/keyrings/signalwire-freeswitch-repo.gpg] https://freeswitch.signalwire.com/repo/deb/debian-release/ `lsb_release -sc` main" >> /etc/apt/sources.list.d/freeswitch.list

# you may want to populate /etc/freeswitch at this point.
# if /etc/freeswitch does not exist, the standard vanilla configuration is deployed
apt-get update && apt-get install -y freeswitch-meta-all


# end-of install-freeswith.sh

Run the above script this will take sometime and install a new freeswith instance



Steps to allow savg to be trunked
docker exec -it freeswitchpbx bash

you can use fs_cli application to issue freeswitch command

- to see the status of sofia sip use "sofia status" command
- to see the status of the internal profile use "sofia status profile internal", this command will provide information regarding websocket address, sip address etc. WS-BIND-URL, 

vi /etc/freeswitch/vars.xml
change 5060 to 5090 5061 to 5091

if call is getting disconnected after 32 seconds
in freeswitch change the below configuration in /etc/freeswitch/sip_profiles/internal.xml

<!-- param name="ext-rtp-ip" value="$${external_rtp_ip}"/ -->
<param name="ext-rtp-ip" value="<<place external ip here>>"/>
<!-- param name="ext-sip-ip" value="$${external_rtp_ip}"/ -->
<param name="ext-sip-ip" value="<<place external ip>>"/>

freeswitch uses 8081 port in /etc/freeswitch/autoload_configs/verto.conf.xml as koreserver also uses 8081 change it to 8084
freeswitch uses 5080 port in /etc/freeswitch/vars.xml as koreserver also uses 5080 change it to 5092, 5083


vi /etc/freeswitch/autoload_configs/acl.conf.xml
Under list tag name="domains" add the below line
<node type="allow" cidr="172.10.0.0/16"/>

Steps to create a new gateway for trunking to save from freeswitch
Under /etc/freeswitch/sip_profiles create a folder with name gateways
Create a file smartassist.xml with the below content:
<gateway name="local.kore.ai">
	<param name="proxy" value="<ip address of the machine>"/>
	<param name="realm" value="local.kore.ai"/>
	<param name="register" value="false"/>
	<param name="context" value="smartassist"/>
	<param name="caller-id-in-from" value="true"/>
</gateway>

Under /etc/freeswitch/dialplan/default.xml add the below block
<extension name="bridge to smartassist">
	    <condition field="destination_number" expression="^10001$">
		    <action application="set" data="rtp_secure_media_inbound=forbidden"/>
		    <action application="set" data="rtp_secure_media_outbound=forbidden"/>
            <action application="bridge" data="sofia/gateway/local.kore.ai/10001"/>
      </condition>
</extension>
In /etc/freeswitch/profiles/sip_profiles/internal.xml add the below XML
<gateways>
    <X-PRE-PROCESS cmd="include" data="gateways/*.xml"/>
</gateways>

Under /etc/freeswitch/directory/default folder create a file 10001.xml and add the below contents
<include>
  <user id="10001">
    <params>
      <param name="password" value="$${default_password}"/>
      <param name="vm-password" value="10001"/>
    </params>
    <variables>
      <variable name="toll_allow" value="local"/>
      <variable name="accountcode" value="10001"/>
      <variable name="effective_caller_id_name" value="Extension 10001"/>
      <variable name="effective_caller_id_number" value="10001"/>
      <variable name="outbound_caller_id_name" value="10001"/>
      <variable name="outbound_caller_id_number" value="10001"/>
      <variable name="callgroup" value="techsupport"/>
    </variables>
  </user>
</include>

In savg portal create a carrier with name local
- Career name = local
- Select a predefined carrier: none
- Add the below ip addresses and ports
        127.0.0.1 5090 
        <internal ip that you get from ifconfig> 5090 check mark both inbound and outbound

Create a sipdevicecall application
- Selectl Applications menu and click "+" button
Application Name: sipdevicecall
Account: default account
Calling webhook: ws://mylocal.kore.ai/korevg/bot?sipdevicecall=true
Enable Use HTTP basic authentication and provide below username and password
Username: kore.ai
Password: zvL7$7$iBqJ42$0@

Call status webhook: ws://mylocal.kore.ai/korevg/bot?sipdevicecall=true
Give same username and password for authentication

Speech synthesis vendor
Google, English (US), Standard-C (Female)

Speech recognizer vendor
Google, English (United States)

Under accounts select "default account"
SIP Realm: local.kore.ai
Application for sip device calls:Select sipdevicecall application from the list
Reistration webhook: http://mylocal.kore.ai/api/internal/korevg/register
Enable  Use HTTP basic authentication and provide below username and password
Username: kore.ai
Password: zvL7$7$iBqJ42$0@

Select speech from the left menu
Click "+" icon
Select vendor as google
Select the google auth json file
Click save button

In smartassist configure voice channel with below details:
Incoming IP address field enter 192.168.29.72,127.0.0.1 <the ip address that you get when you use ifconfig command>

Under DID number enter 10001

Select Default conversational input voice flow, select the start node and select 10001 phone number in choose a phone number drop down

Under Language & Speech menu make sure you select the google for both asr and tts

In API setup menu select a new app with name (Myoutbound) and select SmartAssist Dialout as scope, keep the clientid, secret handy, as we need to generate a JWT token using jwttoken.io website and use it in invoking the dialout public api but there is also an internal api to the dialout api use the apikey header providing the value from koreconfig.json
curl --location 'http://localhost/api/1.1/internal/smartassist/dialout' \
--header 'apikey: yktyderjdzfwhhM3wQkjhsfiaHqSBxTpc4XXOP7v/rHdPYfD6qA9Zc=' \
--header 'X-AMD: true' \
--header 'Content-Type: application/json' \
--data-raw '{
    "bot": "st-781c35f3-cb87-5ed1-a263-796f89a78084",
    "target": "sip:1002@192.168.29.76:5090",
    "caller": "10001"
}'


Use the below bot_korevg configuration in KoreConfig.json
"bot_korevg": {
		"baseAPIURL": "http://localhost:3012/v1",
		"dedicatedHosting": true,
		"endPoints": {
		    "botURLPath": "/korevg/bot",
		    "application": "/Applications",
		    "phoneNo": "/PhoneNumbers",
		    "voipCarrier": "/VoipCarriers",
		    "sipGateway": "/SipGateways",
		    "serviceProvider": "/ServiceProviders"
		},
		"configParams": {
		    "apiKey": "replace the api key",
		    "accountSId": "replace with account sid",
		    "voipCarrierSId": "replace voip carrier sid",
		    "serviceProviderAPIKey" : "replace service provider api key"
		},
		"trunk": "local",
		"sbc": {
		    "koreVGdomainName": "local.kore.ai",
		    "koreVGserverAddress": [
		    "wss://172.10.0.10:8093"
		]
		},
		"botUserSessionTTL": "525600",
		"voice": "en-US-Wavenet-C",
		"language": "en-US"
	
	}



