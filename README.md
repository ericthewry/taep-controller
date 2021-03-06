## TAEP-Controller
This project contains the code for the TAEP Controller.

The platform controller itself is written in [Rust](https://www.rust-lang.org/en-US/), and the data-plane related code is written in [P4](https://p4.org).

The control-plane is using C stubs generated by the P4 compiler to interact with the P4 code and bf_switchd related libraries for all the general interaction with the Tofino platform.

For any additional information regarding TAEP please refer to [github.com/att-innovate/taep](https://github.com/att-innovate/taep).

### Installation
For installation please refer to [TAEP-Scripts](https://github.com/att-innovate/taep-scripts).

### Configuration
At startup by default the controller reads the config file at `config/config.yml`.

A sample config file to configure a simple L2 forwarder forwarding packets between port 0 and 4 using 40G connections:

	bf-bin-path: /root/bf-sde/install
	bf-config-file: /root/taep-controller/p4/l2_switching.conf
	enable-labeling: true
	api-port: 8100
	ports:
        - number: 0
          speed: 40
          autoneg-disabled: true
          fec-disabled: true
        - number: 4
          speed: 40
          autoneg-disabled: true
          fec-disabled: true
	connections:
        - from: 0
          to: 4
          type: bidirectional
	hhd:
        analysis-window-in-seconds: 3
        max-number-of-flows: 200

Parameters:
- **bf-bin-path**: Sets path for bf-sde binaries. Shouldn’t be changed.
- **bf-config-file**: Path to the configuration file `l2_switching.conf` for the P4 code. Shouldn’t be changed.
- **enable-labeling**: If enabling is turned on (default) the controller will write timestamped “labels” for actions related to the divert table in to InfluxDB. Those labels can be used to correlate traffic metrics in InfluxDB to specific IP addresses or ranges.
- **api-port**: The port the embedded REST-based server is listening on. A description of the REST API can be found below.
- **ports**: Configuration of the QSFP ports. Each QSFP port can either be run in split mode 4x10G or single 100G or 40G mode.
	- **number**: Each individual QSFP port has 4 port numbers assigned. Example: Port 0, the first port number assigned to the first QSFP port can either be configured as a 100G, 40G, or 10G port, while port 1-3 can only be configured as 10G port.
	- **speed**: Port speed in Gbits, either 10, 40, or 100.
	- **autoneg-disabled**: Optional argument to overwrite default behavior. By default auto-negotiation on the link level is enabled.
	- **fec-disabled**: Optional argument to overwrite default behavior. By default Forward Error Correction on the link level is turned on for 100G ports and turned off for 10G and 40G ports.
- **connections**: Defines default forwarding behavior. A “connection”, a link is set between two ports, a “from port” and a “to port”. The packet forwarding behavior between those two ports can either be set to unidirectional or bidirectional, in which case packets get forwarded in both directions.
	- **from**: Port-number for one port of the connection.
	- **to**: Port-number for the second port of the connection.
	- **type**: bidirectional - Packets coming in from either port will get forwarded to the other port, unidirectional - Packets get only forwarded from “from port” to the “to port”.
- **hhd**: Settings for the Heavy Hitter Divert functionality.
	- **analysis-window-in-seconds**: defines length of time-window to observe and find Heavy flows.
	- **max-number-of-flows**: Max numbers of flows that get tracked by the TAEP controller.

### Run Controller
For detailed examples on how to use TAEP Controller for network analysis and network experiments please refer to [TAEP-Examples](https://github.com/att-innovate/taep/blob/master/EXAMPLES.md).

Start TAEP controller:

	$ service taep start

Check Logs:

	$ cd /root/openleaf-barefoot-scripts
	$ ./scripts/log-taep.sh

Use `ctrl-c` to interrupt.

Following log output indicates that the controller has start successfully:

	Added entry to Forwarding Table, Handle 1
	Added entry to Forwarding Table, Handle 2
	HHD Max Number of Flow set to 200
	HHD Analysis Window 3s

To stop the TAEP controller:

	$ service taep stop

Be aware that this only shuts down the controller. The P4 code previously loaded on to the Tofino ASIC will still be processing packets based on the last active configuration.

### REST-API
The TAEP controller offers an extensive REST API to automate and control experiments and traffic analysis.

#### `/admin/ping`
Simple request to check if controller is up.

Request

	$ curl http://localhost:8100/admin/ping

Response

	pong

#### `/metrics`
Retrieve network traffic data from all active ports.

Request

	$ curl http://localhost:8100/metrics

Response

	[{"chassis_port":0,"packets_in":85372472112,"packets_out":8361430405,"octets_in":122773863830132,"octets_out":621824672499,"packets_dropped_buffer_full":0} ...}

#### `/divert/dest`  and `/divert/src`
Set or update rules to divert packets for a certain source or destination IP address or range through a different path.

Example: Divert TCP/UDP packets with source address 198.32.44.22/32 incoming at port 0 to port 16.

	$ curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{"port_ingress": 0, "port_egress": 16, "ip_address": "198.32.44.22", "ip_prefix_length": 32}' 'http://localhost:8100/divert/src'

Response

	{"result":"done"}

Example: Divert TCP/UDP packets with destination address 198.0.0.0/8 incoming at port 4 out to port 12

	$ curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{"port_ingress": 4, "port_egress": 12, "ip_address": "198.0.0.0", "ip_prefix_length": 8}' 'http://localhost:8100/divert/dest'

Example: Overwrite **all** existing Divert rules with a new one, diverting TCP/UDP packets with destination address 198.32.44.23/32 incoming at port 0 to port 16.

	$ curl -X PATCH --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{"port_ingress": 0, "port_egress": 16, "ip_address": "198.32.44.23", "ip_prefix_length": 32}' 'http://localhost:8100/divert/dest'

#### `/divert`
Reset, delete all existing Divert rules.

Request:

	$ curl -X DELETE 'http://localhost:8100/divert'

Response:

	{"result":"done"}

#### `/flows`
Either enable Flow learning for a defined period of time or retrieve the learned flows.

Example: Enable Flow Learning on port 4 for 5 seconds and set the max number of learned flows to 500.

	$ curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{"port_ingress": 4, "max_number_of_flows": 500, "time_window_in_seconds": 5}' 'http://localhost:8100/flows'

Response:

	{"result":"done"}

Example: Retrieve the learned flows.

	$ curl http://localhost:8100/flows

Response

	[{"src_addr":"10.250.3.24","src_addr_int":184156952,"src_port":22,"dst_addr":"10.250.3.25","dst_addr_int":184156953,"dst_port":60338,"ipv4_protocol":6,"hash1":12283,"hash2":8288},{"src_addr":"91.189.89.198","src_addr_int":1539135942,"src_port":123,"dst_addr":"10.250.3.25","dst_addr_int":184156953,"dst_port":123,"ipv4_protocol":17,"hash1":10700,"hash2":14037} ....]

#### `/hhd/dest` and `/hhd/src`
Manage the implemented Heavy Hitter Divert functionality.

In each time window the system resolves the flow the “Heavy Hitter”, in our case the flow with the most number of packets. For the next time window this flow will then be diverted through a different port and path.

Example: For this example all the traffic by default is flowing 4 -> 12 -> 8. A eavy Hitter has to be diverted through port 4 -> 20 -> 16. The packets get counted at port 8 and 16. We measure all the flows based on their source addresses.

	$ curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{"port_ingress": 8, "port_ingress_divert": 16, "divert_ingress": 4, "divert_egress": 20}' 'http://localhost:8100/hhd/src'

Response

	{"result":"done"}

#### `/hhd`

Reset, delete an active Heave Hitter Divert rule.

Request

	$ curl -X DELETE 'http://localhost:8100/hhd'

Response

	{"result":"done"}
