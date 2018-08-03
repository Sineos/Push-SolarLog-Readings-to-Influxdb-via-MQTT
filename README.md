
This flow receives a json string from SolarLog PV Monitoring units containing all available measurements.
These measurements are then transformed in a function-node to be send to influxdb's telegraf via the mqtt protocol.

SolarLog will update the JSON string every 15 seconds. So it makes no sense to pull it more often. The function node that parses the JSON string will make sure that only updated values are processed.  

# How To

## Prerequisites

 - [NodeRed](http://nodered.org/)
 - [Mosquitto](http://mosquitto.org/) or any other MQTT message broker
 - [Telegraf](https://influxdata.com/time-series-platform/telegraf/)
 - [InfluxDB](https://influxdata.com/time-series-platform/influxdb/)
 - [Grafana](http://grafana.org/) Optional for visualization

Setup instructions are widely available in the WWW for these products and may vary according to the hardware (RaspberryPI, i386 etc) and linux distribution.

## Configuration

### NodeRed

#### Import Flow

Copy the flow.json in raw format to your clipboard.
In NodeRed import the copied flow:
![import_flow](https://raw.githubusercontent.com/Sineos/Push-Volkszaehler-Readings-to-Influxdb-via-MQTT/master/src_readme/import_nodered.jpg)

#### Configure POST to SolarLog

![enter image description here](https://raw.githubusercontent.com/Sineos/Push-SolarLog-Readings-to-Influxdb-via-MQTT/master/src_readme/edit_request_node.jpg)

Modify the URL to point to your SolarLog IP. The path needs to be:

    IP_Solarlog/getjp

#### Parse SolarLog JSON

Double-click the function node `Parse SolarLog JSON`

![enter image description here](https://raw.githubusercontent.com/Sineos/Push-SolarLog-Readings-to-Influxdb-via-MQTT/master/src_readme/edit_function_node.jpg)

 - `Identifier` is the key in the SolarLog JSON string that identifies a certain measurement. **Do not edit this value**.
 - `Active` can be either set to 0 or 1 and will control, whether this `Identifier` is processed and send via MQTT. Set to 1 for every value, you want to store in InfluxDB. 
 - The `topic` is needed for the MQTT message broker. This reflects the "path" or "channel" under which the MQTT message broker will publish the data. This will be the channel that we use in Telegraf to collect the data
 - `Measurement` is needed to put the data into the InfluxDB and will be used along with the `tags`. Read the InfluxDB manual [here](https://docs.influxdata.com/influxdb/v0.13/concepts/key_concepts/) and [here](https://docs.influxdata.com/influxdb/v0.13/concepts/schema_and_data_layout/) for a better understanding of the measurement and tag concept.
 - `Tags` are to be used alongside the `measurement` and need to be specified in the format `Tag_Key = Tag_Value`. They can be used to sort, query and cluster data within InfluxDB. You can specify none to an arbitrary number of tags. 

#### Configure MQTT output

![enter image description here](https://raw.githubusercontent.com/Sineos/Push-Volkszaehler-Readings-to-Influxdb-via-MQTT/master/src_readme/edit_mqtt.jpg)

Edit the `Server` to match the IP and port of the MQTT message broker (e.g. Mosquitto).

### Telegraf

In the telegraf.conf file you have to specify an Input-Plugin for the MQTT protocol:

    # Read metrics from MQTT topic(s)
    [[inputs.mqtt_consumer]]
      servers = ["localhost:1883"]
      ## MQTT QoS, must be 0, 1, or 2
      qos = 0
    
      ## Topics to subscribe to
      topics = [
        "/power/#",
      ]
    
      # if true, messages that can't be delivered while the subscriber is offline
      # will be delivered when it comes back (such as on service restart).
      # NOTE: if true, client_id MUST be set
      persistent_session = false
      # If empty, a random client ID will be generated.
      client_id = ""
    
      ## username and password to connect MQTT server.
      # username = "telegraf"
      # password = "metricsmetricsmetricsmetrics"
    
      ## Optional SSL Config
      # ssl_ca = "/etc/telegraf/ca.pem"
      # ssl_cert = "/etc/telegraf/cert.pem"
      # ssl_key = "/etc/telegraf/key.pem"
      ## Use SSL but skip chain & host verification
      # insecure_skip_verify = false
    
      ## Data format to consume.
      ## Each data format has it's own unique set of configuration options, read
      ## more about them here:
      ## https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md
      data_format = "influx"

 - Modify `servers` to match the IP and port of the MQTT message broker (e.g. Mosquitto) 
 - Modify `topics` to the topic(s) you used in the function node of the flow. Note that the `#` will match anything after the first part of the path
