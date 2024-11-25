# Helm chart for chirpstack 4

Complete settings with default values
```yaml
_common:
  namespace: "chirpstack"
  region:
    - name: "eu868" # name on region to enable for the chart to select the correct settings file
      settings: 
        gateway:
          private : false  # If enabled, gateways can only be used by devices under the same tenant.
          enabledBackend: "mqtt" # The enabled backend type.
          installationMargin: 10  # Installation margin (dB) used by the ADR engine.
          # A higher number means that the network-server will keep more margin,
          # resulting in a lower data-rate but decreasing the chance that the
          # device gets disconnected because it is unable to reach one of the
          # surrounded gateways.
          rxWindow: 0  # RX window (Class-A).
          # Set this to:
          # 0: RX1 / RX2
          # 1: RX1 only
          # 2: RX2 only
          rx1Delay: 1 # RX1 delay (1 - 15 seconds).
          rx1DrOffset: 0 # RX1 data-rate offset
          rx2Dr: 0 # RX2 data-rate
          rx2PreferOnRx1DrLt: 0 # Prefer RX2 on RX1 data-rate less than.
          # Prefer RX2 over RX1 based on the RX1 data-rate. When the RX1 data-rate
          # is smaller than the configured value, then the Network Server will
          # first try to schedule the downlink for RX2, failing that (e.g. the gateway
          # has already a payload scheduled at the RX2 timing) it will try RX1.
          rx2PreferOnLinkBudget: false 
          # Prefer RX2 on link budget.
          # When the link-budget is better for RX2 than for RX1, the Network Server will first
          # try to schedule the downlink in RX2, failing that it will try RX1.
          downlinkTxPower: -1 # Downlink TX Power (dBm)
          # When set to -1, the downlink TX Power from the configured band will
          # be used.
          # Please consult the LoRaWAN Regional Parameters and local regulations
          # for valid and legal options. Note that the configured TX Power must be
          # supported by your gateway(s).
          adrDisabled: false # ADR is disabled.
          minDr: 0 # Minimum data-rate.
          max_dr: 5 # Maximum data-rate.
        network: 
          rejoinRequest:
            enabled: false # Request devices to periodically send rejoin-requests.
            maxCount: 0 # The device must send a rejoin-request type 0 at least every 2^(max_count_n + 4) uplink messages. Valid values are 0 to 15.
            maxTime: 0 # The device must send a rejoin-request type 0 at least every 2^(max_time_n + 10) seconds. Valid values are 0 to 15. # 0  = roughly 17 minutes. 15 = about 1 year
          classB:
            pingSlotDr: 0 # Ping-slot data-rate. # set this to 0 to use the default frequency plan for the configured region (which could be frequency hopping).
            pingSlotFrequency: 0 # Ping-slot frequency (Hz). 
  #  - name: "eu433"
  netId: "000000" # Network identifier (NetID, 3 bytes) encoded as HEX (e.g. 010203).
  secret: "iLikePink" # This secret is used for generating login and API tokens, make sure this
  # is never exposed. Changing this secret will invalidate all login and API
  # tokens. The following command can be used to generate a random secret:
  #   openssl rand -base64 32
  logging: "idle"   # Options are: trace, debug, info, warn error.
  apiPort: "8080"   # port to bind the API interface to.
  integration: "mqtt" # integration to enable for chirpstack
  eventTopic: "application/{{application_id}}/device/{{dev_eui}}/event/{{event}}" # Event topic template.
  commandTopic: "application/{{application_id}}/device/{{dev_eui}}/command/{{command}}" # Command topic template. 
  mqtt:
    host: broker.local
    port: 1883
    username: user
    password: pass
    qos: 2
    cleanSession: false
    clientId: "chirpstack"
    caCert: "none"
    tlsCert: "none"
    tlsKey: "none"
    keepAliveInterval: "30s" # This defines the maximum time that that should pass without communication between the client and server.

  postgres: 
    settings: # common settings for postgres
      postgresUser: chirpstack2
      dbName: chirpstack2
      role: chirpstack2
      password: chirpstack2
      sslmode: "disable"
      # SSL mode options:
      #  * disable - no SSL
      #  * require - Always SSL (skip verification)
      #  * verify-ca - Always SSL (verify that the certificate presented by the server was signed by a trusted CA)
      #  * verify-full - Always SSL (verify that the certification presented by the server was signed by a trusted CA and the server host name matches the one in the certificate)
      caCert: "" # CA certificate (optional). Set this to the path of the CA certificate in case you are using TLS and the server-certificate is not signed by a CA in the platform certificate store.
      maxOpenConnections: 10 # This sets the max. number of open connections that are allowed in the PostgreSQL connection pool.
      minIdleConnections: 0 # This sets the min. number of idle connections in the PostgreSQL connection pool (0 = equal to max_open_connections).
    external: 
      enabled: true # if enabled, host and port needs to point to external postgres, otherwise local postgres will be deployed and used
      host: "gres.local"
      port: 1234
  redis: 
    settings:
      tls: false
      cluster : false 
    external:
      enabled: false # if enabled, host and port needs to point to external redis, otherwise local redis will be deployed and used
      host: "local.redis"
      port: 6379  


chirpstack:
  replicaCount: 1
  image:
    repository: chirpstack/chirpstack
    pullPolicy: IfNotPresent
    tag: "4"
  args: 
    - "-c"
    - "/etc/chirpstack"
  service:
    port: 8080
    type: ClusterIP

  volumeMounts: 
    name: configuration
    mountPath: /etc/chirpstack

  volumes: 
    name: configuration
    configMap:
      name: chirpstack-configuration
  config: |
    # extra config fields not covered in common settings.
api:
  image:
    repository: chirpstack/chirpstack-rest-api
    pullPolicy: IfNotPresent
    tag: "4"
  args: 
    - "--server"
    - "chirpstack:8080"
    - "--bind"
    - "0.0.0.0:8090"
    - "--insecure"
  service:
    port: 8090
    type: ClusterIP

gb:  
  image:
    repository: chirpstack/chirpstack-gateway-bridge
    pullPolicy: IfNotPresent
    tag: "4"
  volumeMounts:
    mountPath: /etc/chirpstack-gateway-bridge
  volumes:
    configMap:
      name: chirpstack-gateway-bridge-eu868-configuration
  service:
    port: 1700
    type: ClusterIP
  useCommonMqttSettings: true # mqtt values are populated in the config automatically from common mqtt settings, otherwise it needs to be specified in the config section.
  config:
    general:
      logLevel: 4
      logJson: false
      logToSyslog: false
    filters:
      netIds: []
      joinEuis: []
    backend:
      type: "semtech_udp"
      semtechUdp:
        udpBind: "0.0.0.0:1700"
        fakeRxTime: false
      concentratord:
        eventUrl: "ipc:///tmp/concentratord_event"
        commandUrl: "ipc:///tmp/concentratord_command"
      basicStation:
        bind: ":3001"
        tlsCert: ""
        tlsKey: ""
        caCert: ""
        statsInterval: "30s"
        pingInterval: "1m0s"
        timesyncInterval: "0s"
        readTimeout: "1m5s"
        writeTimeout: "1s"
    integration:
      marshaler: "protobuf"
      mqtt:
        eventTopicTemplate: "gateway/{{ .GatewayID }}/event/{{ .EventType }}"
        stateTopicTemplate: "gateway/{{ .GatewayID }}/state/{{ .StateType }}"
        commandTopicTemplate: "gateway/{{ .GatewayID }}/command/#"
        stateRetained: true
        keepAlive: "30s"
        maxReconnectInterval: "1m0s"
        terminateOnConnectError: false
        auth:
          type: "generic"
          generic:
            servers:
              - "tcp://127.0.0.1:1883"
            username: ""
            password: ""
            qos: 0
            cleanSession: true
            clientId: ""
            caCert: ""
            tlsCert: ""
            tlsKey: ""
          gcpCloudIotCore:
            server: "ssl://mqtt.googleapis.com:8883"
            deviceId: ""
            projectId: ""
            cloudRegion: ""
            registryId: ""
            jwtExpiration: "24h0m0s"
            jwtKeyFile: ""
          azureIotHub:
            deviceConnectionString: ""
            sasTokenExpiration: "24h0m0s"
            deviceId: ""
            hostname: ""
            tlsCert: ""
            tlsKey: ""
    metrics:
      prometheus:
        endpointEnabled: false
        bind: ""
    metaData:
      static: |
        # example:
        # serial_number="A1B21234"
      dynamic:
        executionInterval: "1m0s"
        maxExecutionDuration: "1s"
        splitDelimiter: "="
        commands: |
          # Example:
          # temperature="/opt/gateway-temperature/gateway-temperature.sh"
    commands: |
      # Example:
      # [commands.commands.reboot]
      # max_execution_duration="1s"
      # command="/usr/bin/reboot"
  configExtra: |
    # use this to add extra settings to the config

postgres:
  image:
    repository: postgres
    pullPolicy: IfNotPresent
    tag: "14-alpine"
  service:
    port: 5432
    type: ClusterIP
  volumeMounts: 
    name: data
    mountPath: /var/lib/postgresql/data
  volumes: 
    initdb:
      configMap:
        name: postgres-initdb
  volumeClaim:
    resources:
        requests:
          storage: 500Mi
      
redis:
  image:
    repository: redis
    pullPolicy: IfNotPresent
    tag: "7-alpine"
  service:
    port: 6379
    type: ClusterIP
  volumeMounts: 
    name: data
    mountPath: /data
  volumeClaim:
    resources:
        requests:
          storage: 100Mi
```