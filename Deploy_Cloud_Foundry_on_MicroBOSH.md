### Deploy Cloud Foundry on MicroBOSH

#### Upload a Release

Upload the source code and related resources of Cloud Foundry to your MicroBOSH. The cf-release repository has everything you need.

```sh
# Clone the repository
git clone https://github.com/cloudfoundry/cf-release.git ~/cf-release
cd cf-release
# Upload the latest release (161 at the moment)
BUNDLE_GEMFILE=~/bosh/Gemfile bundle exec bosh upload release releases/cf-161.yml
```

This step takes some time.

#### Upload a Stemcell

Upload a stemcell to your MicroBOSH. You can use the same stemcell file which you used for bootstraping your MicroBOSH.

```sh
BUNDLE_GEMFILE=~/bosh/Gemfile bundle exec bosh upload stemcell <path_to_stemcell>
```


#### Describe Manifest

Create your deployment manifest with the template below and save it whith your prefered name e.g. `cf.yml`.

```yaml
---
name: cf
director_uuid: 884aab78-3b73-494c-aa6f-b7fe9b2d7e1b # UUID shown by the bosh status command

releases:
 - name: cf
   version: 161 # Verison number of the uploded release

networks:
- name: default
  type: dynamic
  cloud_properties:

    # Only for Basic Zone users
    security_groups:
      - bosh # Securiy group which opens all TCP and UDP ports

    # Only for Advanced Zone users
    network_name: <network_name> # subnetwork

# Only for Advanced Zone users
# Network with floating IP addresses
- name: floating
  type: vip
  cloud_properties: {}


compilation:
  workers: 6
  network: default
  reuse_compilation_vms: true
  cloud_properties:
    instance_type: m1.medium        # VM type
    ephemeral_volume: Datadisk 40GB # Data disk offering name of additonal disk

update:
  canaries: 1
  canary_watch_time: 30000-60000
  update_watch_time: 30000-60000
  max_in_flight: 4

resource_pools:
  - name: small
    network: default
    size: 8
    stemcell:
      name: bosh-cloudstack-kvm-ubuntu
      version: latest
    cloud_properties:
      instance_type: m1.small        # VM type
      ephemeral_volume: Datadisk 40GB # Data disk offering name of additonal disk

  - name: large
    network: default
    size: 1
    stemcell:
      name: bosh-cloudstack-kvm-ubuntu
      version: latest
    cloud_properties:
      instance_type: m1.large        # VM type
      ephemeral_volume: Datadisk 40GB # Data disk offering name of additional disk

jobs:
  - name: nats
    release: cf
    template:
      - nats
    instances: 1
    resource_pool: small
    networks:
      - name: default
        default: [dns, gateway]

  - name: syslog_aggregator
    release: cf
    template:
      - syslog_aggregator
    instances: 1
    resource_pool: small
    persistent_disk: 65536
    networks:
      - name: default
        default: [dns, gateway]

  - name: postgres
    release: cf
    template:
      - postgres
    instances: 1
    resource_pool: small
    persistent_disk: 65536
    networks:
      - name: default
        default: [dns, gateway]
    properties:
      db: databases

  - name: nfs_server
    release: cf
    template:
      - debian_nfs_server
    instances: 1
    resource_pool: small
    persistent_disk: 65536
    networks:
      - name: default
        default: [dns, gateway]

  - name: uaa
    release: cf
    template:
      - uaa
    instances: 1
    resource_pool: small
    networks:
      - name: default
        default: [dns, gateway]

  - name: cloud_controller
    release: cf
    template:
      - cloud_controller_ng
    instances: 1
    resource_pool: small
    networks:
      - name: default
        default: [dns, gateway]
    properties:
      ccdb: ccdb

  - name: router
    release: cf
    template:
      - gorouter
    instances: 1
    resource_pool: small
    networks:
      - name: default
        default: [dns, gateway]

      # Only for Advanced zone users
      # You can set floating addresses to jobs
      # Acquire Public IP addresses on your Web UI before deploying
      # (Don't remove `default` network above even if `floating` is added)
      - name: floating
        static_ips:
          - <IP address for Router>

  - name: health_manager
    release: cf
    template:
      - health_manager_next
    instances: 1
    resource_pool: small
    networks:
      - name: default
        default: [dns, gateway]

  - name: dea
    release: cf
    template: dea_next
    instances: 1
    resource_pool: large
    networks:
      - name: default
        default: [dns, gateway]

properties:

  domain: your.domain.name # replace these values with your domain name
  system_domain: your.domain.name
  system_domain_organization: your.domain.name
  app_domains:
    - your.domain.name

  networks:
    apps: default
    management: default

  nats:
    address: 0.nats.default.cf.microbosh
    port: 4222
    user: nats
    password: c1oudc0w
    authorization_timeout: 5

  router:
    port: 8081
    status:
      port: 8080
      user: gorouter
      password: c1oudcow

  dea: &dea
    memory_mb: 2048
    disk_mb: 20000
    directory_server_protocol: http

  dea_next: *dea

  syslog_aggregator:
    address: 0.syslog-aggregator.default.cf.microbosh
    port: 54321

  nfs_server:
    address: 0.nfs-server.default.cf.microbosh
    network: "*.cf.microbosh"
    idmapd_domain: your.domain.name

  debian_nfs_server:
    no_root_squash: true

  databases: &databases
    db_scheme: postgres
    address: 0.postgres.default.cf.microbosh
    port: 5524
    roles:
      - tag: admin
        name: ccadmin
        password: c1oudc0w
      - tag: admin
        name: uaaadmin
        password: c1oudc0w
    databases:
      - tag: cc
        name: ccdb
        citext: true
      - tag: uaa
        name: uaadb
        citext: true

  ccdb: &ccdb
    db_scheme: postgres
    address: 0.postgres.default.cf.microbosh
    port: 5524
    roles:
      - tag: admin
        name: ccadmin
        password: c1oudc0w
    databases:
      - tag: cc
        name: ccdb
        citext: true

  ccdb_ng: *ccdb

  uaadb:
    db_scheme: postgresql
    address: 0.postgres.default.cf.microbosh
    port: 5524
    roles:
      - tag: admin
        name: uaaadmin
        password: c1oudc0w
    databases:
      - tag: uaa
        name: uaadb
        citext: true

  cc_api_version: v2

  cc: &cc
    logging_level: debug
    external_host: api
    srv_api_uri: http://api.your.domain.name
    cc_partition: default
    db_encryption_key: c1oudc0w
    bootstrap_admin_email: admin@your.domain.name
    bulk_api_password: c1oudc0w
    uaa_resource_id: cloud_controller
    staging_upload_user: uploaduser
    staging_upload_password: c1oudc0w
    resource_pool:
      resource_directory_key: cc-resources
      # Local provider when using NFS
      fog_connection:
        provider: Local
    packages:
      app_package_directory_key: cc-packages
    droplets:
      droplet_directory_key: cc-droplets
    default_quota_definition: runaway

  ccng: *cc

  login:
    enabled: false

  uaa:
    url: http://uaa.your.domain.name
    spring_profiles: postgresql
    no_ssl: true
    catalina_opts: -Xmx768m -XX:MaxPermSize=256m
    resource_id: account_manager
    jwt:
      signing_key: |
        -----BEGIN RSA PRIVATE KEY-----
        MIICXAIBAAKBgQDHFr+KICms+tuT1OXJwhCUmR2dKVy7psa8xzElSyzqx7oJyfJ1
        JZyOzToj9T5SfTIq396agbHJWVfYphNahvZ/7uMXqHxf+ZH9BL1gk9Y6kCnbM5R6
        0gfwjyW1/dQPjOzn9N394zd2FJoFHwdq9Qs0wBugspULZVNRxq7veq/fzwIDAQAB
        AoGBAJ8dRTQFhIllbHx4GLbpTQsWXJ6w4hZvskJKCLM/o8R4n+0W45pQ1xEiYKdA
        Z/DRcnjltylRImBD8XuLL8iYOQSZXNMb1h3g5/UGbUXLmCgQLOUUlnYt34QOQm+0
        KvUqfMSFBbKMsYBAoQmNdTHBaz3dZa8ON9hh/f5TT8u0OWNRAkEA5opzsIXv+52J
        duc1VGyX3SwlxiE2dStW8wZqGiuLH142n6MKnkLU4ctNLiclw6BZePXFZYIK+AkE
        xQ+k16je5QJBAN0TIKMPWIbbHVr5rkdUqOyezlFFWYOwnMmw/BKa1d3zp54VP/P8
        +5aQ2d4sMoKEOfdWH7UqMe3FszfYFvSu5KMCQFMYeFaaEEP7Jn8rGzfQ5HQd44ek
        lQJqmq6CE2BXbY/i34FuvPcKU70HEEygY6Y9d8J3o6zQ0K9SYNu+pcXt4lkCQA3h
        jJQQe5uEGJTExqed7jllQ0khFJzLMx0K6tj0NeeIzAaGCQz13oo2sCdeGRHO4aDh
        HH6Qlq/6UOV5wP8+GAcCQFgRCcB+hrje8hfEEefHcFpyKH+5g1Eu1k0mLrxK2zd+
        4SlotYRHgPCEubokb2S1zfZDWIXW3HmggnGgM949TlY=
        -----END RSA PRIVATE KEY-----
      verification_key: |
        -----BEGIN PUBLIC KEY-----
        MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDHFr+KICms+tuT1OXJwhCUmR2d
        KVy7psa8xzElSyzqx7oJyfJ1JZyOzToj9T5SfTIq396agbHJWVfYphNahvZ/7uMX
        qHxf+ZH9BL1gk9Y6kCnbM5R60gfwjyW1/dQPjOzn9N394zd2FJoFHwdq9Qs0wBug
        spULZVNRxq7veq/fzwIDAQAB
        -----END PUBLIC KEY-----
    cc:
      client_secret: c1oudc0w
    admin:
      client_secret: c1oudc0w
    batch:
      username: batchuser
      password: c1oudc0w
    client:
      autoapprove:
        - cf
    clients:
      cf:
        override: true
        authorized-grant-types: password,implicit,refresh_token
        authorities: uaa.none
        scope: cloud_controller.read,cloud_controller.write,openid,password.write,cloud_controller.admin,scim.read,scim.write
        access-token-validity: 7200
        refresh-token-validity: 1209600
    scim:
      users:
      - admin|c1oudc0w|scim.write,scim.read,openid,cloud_controller.admin
      - services|c1oudc0w|scim.write,scim.read,openid,cloud_controller.admin
```
