#Setup a Full BOSH (Optional)

##Sample Templates
As CloudStack basic zone does not support VIP networks and Floating IPs. You need some tricks to deploy releases which require floating IPs such as Full BOSH when you use CloudStack basic zone.

For Full BOSH, you need two separate manifest files. One for DNS and another for other jobs.

###bosh-dns.yml

```yaml
---
name: bosh-dns
director_uuid: <Director UUID>

release:
  name: bosh
  version: latest

compilation:
  workers: 3
  network: default
  reuse_compilation_vms: true
  cloud_properties:
    instance_type: m1.small

update:
  canaries: 1
  canary_watch_time: 3000-120000
  update_watch_time: 3000-120000
  max_in_flight: 4
  max_errors: 1

networks:
  - name: default
    type: dynamic
    cloud_properties:
      security_groups:
        - bosh  # Security group

resource_pools:
  - name: small
    network: default
    size: 1
    stemcell:
      name: bosh-cloudstack-kvm-ubuntu
      version: latest
    cloud_properties:
      instance_type: m1.small

jobs:
  - name: powerdns
    template:
      - powerdns
    instances: 1
    resource_pool: small
    persistent_disk: 40960
    networks:
      - name: default
        default: [dns, gateway]

properties:
  env:

  cloudstack:
    endpoint: <endpoint_url>
    api_key: <app_key>
    secret_access_key: <secret_access_key>
    default_key_name: <ssh_key_name>
    default_security_groups:
      - bosh # Security group
    state_timeout: 600
    stemcell_public_visibility: true
    default_zone: <zone>

  postgres: &bosh_db
    host: 0.postgres.default.bosh.microbosh
    password: bosh
    database: bosh
    user: bosh

  dns:
    address: 198.51.100.89 # Put a random IP address first, and replace the correct address later
    db: *bosh_db
    recursor: 198.51.100.39 # IP address of MicroBOSH
```

###bosh.yml

```yaml
---
name: bosh
director_uuid: <Director UUID>

release:
  name: bosh
  version: latest

compilation:
  workers: 3
  network: default
  reuse_compilation_vms: true
  cloud_properties:
    instance_type: m1.small

update:
  canaries: 1
  canary_watch_time: 3000-120000
  update_watch_time: 3000-120000
  max_in_flight: 4
  max_errors: 1

networks:
  - name: default
    type: dynamic
    cloud_properties:
      security_groups:
        - bosh

resource_pools:
  - name: small
    network: default
    size: 6
    stemcell:
      name: bosh-cloudstack-kvm-ubuntu
      version: latest
    cloud_properties:
      instance_type: m1.small

  - name: medium
    network: default
    size: 1
    stemcell:
      name: bosh-cloudstack-kvm-ubuntu
      version: latest
    cloud_properties:
      instance_type: m1.medium

jobs:
  - name: nats
    template: nats
    instances: 1
    resource_pool: small
    networks:
      - name: default
        default: [dns, gateway]

  - name: postgres
    template: postgres
    instances: 1
    resource_pool: small
    persistent_disk: 2048
    networks:
      - name: default
        default: [dns, gateway]

  - name: redis
    template: redis
    instances: 1
    resource_pool: small
    networks:
      - name: default
        default: [dns, gateway]

  - name: director
    template: director
    instances: 1
    resource_pool: medium
    persistent_disk: 4096
    networks:
      - name: default
        default: [dns, gateway]

  - name: blobstore
    template: blobstore
    instances: 1
    resource_pool: small
    networks:
      - name: default
        default: [dns, gateway]

  - name: registry
    template: registry
    instances: 1
    resource_pool: small
    networks:
      - name: default
        default: [dns, gateway]

  - name: health_monitor
    template: health_monitor
    instances: 1
    resource_pool: small
    networks:
      - name: default
        default: [dns, gateway]

properties:
  env:

  cloudstack:
    endpoint: <endpoint_url>
    api_key: <app_key>
    secret_access_key: <secret_access_key>
    default_key_name: <ssh_key_name>
    default_security_groups:
      - bosh # Security group
    state_timeout: 600
    stemcell_public_visibility: true
    default_zone: <zone>

  nats:
    address: 0.nats.default.bosh.microbosh
    user: nats
    password: nats

  postgres: &bosh_db
    host: 0.postgres.default.bosh.microbosh
    password: bosh
    database: bosh
    user: bosh

  dns:
    address: <ip_address_of_dns_job> # The IP address of the VM deployed with bosh-dns.yml
    db: *bosh_db
    recursor: 198.51.100.39 # IP address of MicroBOSH

  redis:
    address: 0.redis.default.bosh.microbosh
    password: redis

  director:
    name: bosh
    address: 0.director.default.bosh.microbosh
    db: *bosh_db

  blobstore:
    address: 0.blobstore.default.bosh.microbosh
    agent:
      user: agent
      password: agent
    director:
      user: director
      password: director

  registry:
    address: 0.registry.default.bosh.microbosh
    db:
      host: 0.postgres.default.bosh.microbosh
      database: bosh
      password: bosh
    http:
      user: registry
      password: registry

  hm:
    http:
      user: hm
      password: hm
    director_account:
      user: admin
      password: admin
    event_nats_enabled: false
    email_notifications: false
    tsdb_enabled: false
    pagerduty_enabled: false
    varz_enabled: true
```
### Deployment Procedure

1. Deploy bosh-dns.yml
2. Put the IP address of the deployed VM to bosh.yml and bosh-dns.yml
3. Deploy bosh.yml
4. Redeploy bosh-dns.yml

You can find the IP address of deployed VMs by the `bosh status` command.
