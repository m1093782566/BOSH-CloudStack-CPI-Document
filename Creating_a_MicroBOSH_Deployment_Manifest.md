#Creating a MicroBOSH Deployment Manifest

##Example Manifests
These examples are designed to illuminate what each IaaS requires in a MicroBOSH deployment manifest.

###CloudStack
You can adapt this example to your environment settings.
You need some settings about your MicroBOSH deployment. Create new directories (usually `deployments` and `deployments/firstbosh`) and write your manifest file using the template below. The manifest file must be saved with the file name `micro_bosh.yml` and in the child directoriy you created.

```sh
mkdir -p ~/deployments/firstbosh
vi ~/deployments/firstbosh/micro_bosh.yml
```

```yaml
---
name: firstbosh
logging:
  level: DEBUG
network:
  type: dynamic

  # Only for Advanced Zone users. Delete these lines on Basic Zone
  vip: <static_ip> # Optional. Public IP address for MicroBOSH. Delete this line if not needed
  cloud_properties:
    network_name: <network_name> # Subnetwork name (not VPC name)

resources:
  persistent_disk: 40960
  cloud_properties:
    instance_type: <instance_type> # VM type
cloud:
  plugin: cloudstack
  properties:
    cloudstack:
      endpoint: <your_end_point_url> # Ask for your administrator
      api_key: <your_api_key> # You can find at your user page
      secret_access_key: <your_secret_access_key> # Same as above
      default_key_name: <default_keypair_name> # Your keypair name (see the next section)
      private_key: <path_to_your_private_key> # The path to the private key file of your key pair
      state_timeout: 600
      state_timeout_volume: 1200
      stemcell_public_visibility: true
      default_zone: <default_zone_name> # Zone name of your instaption server

      # Only for Basic Zone users. Delete these lines on Advanced Zone
      default_security_groups:
        - <security_groups_for_bosh> # Security group name which opens all TCP and UDP port

    registry:
      endpoint: http://admin:admin@<ip_address_of_your_inception_sever>:25889 # Use `ifconfig` to confirm
      user: admin
      password: admin
```

###CloudStack SSH Key Pair
If you have no SSH key Pair registered to your CloudStack, you need generate one to enable SSH access to VM created by BOSH from the Director and CLI.
If your web console does not provide an interface to generate key pairs. Use the script below:
```ruby
#!/usr/bin/env ruby
require 'yaml'
require 'uri'
require 'fog'

settings = YAML.load(File.read(ARGV[0]))
name = ARGV[1]

endpoint_uri = URI.parse(settings['cloud']['properties']['cloudstack']['endpoint'])

fog_params = {
  :provider => 'CloudStack',
  :cloudstack_api_key => settings['cloud']['properties']['cloudstack']['api_key'],
  :cloudstack_secret_access_key => settings['cloud']['properties']['cloudstack']['secret_access_key'],
  :cloudstack_scheme => endpoint_uri.scheme,
  :cloudstack_host => endpoint_uri.host,
  :cloudstack_port => endpoint_uri.port,
  :cloudstack_path => endpoint_uri.path,
}

client = Fog::Compute.new(fog_params)

response = client.create_ssh_key_pair(name)
puts response['createsshkeypairresponse']['keypair']['privatekey']
```

You can run this script like:

```
# <script_name> <path_to_micro_bosh_yml> <key_pair_name>
./gen_key_pair ~/deployments/firstbosh/micro_bosh.yml bosh_test
```

And as script shows the private key, save it to the file path specifyed at `private_key` in `micro_bosh.yml`.

###CloudStack networking options
If you are using basic zone, the network block takes the following form:

```yaml
network:
  type: dynamic
    cloud_properties:
        network_name: <network_name>
```

If you are using advanced zone, the network block takes the following form:
```yaml
network:
  type: dynamic
    vip: <static_ip> # Public IP address for MicroBOSH.
      cloud_properties:
          network_name: <network_name>
```

###CloudStack cloud properties
Some CloudStack cloud properties are explained in the manifest file using comments. You should pay attention to the default_security_groups field in the manifest file as it is only supported in basic zone. You needn't to configure it when you use advanced zone.
