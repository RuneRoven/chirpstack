# Helm chart for chirpstack 4
to add repo use: https://runeroven.github.io/chirpstack/

Complete settings with default values
The current Values file consists of a minimal setup.  
Add the relevant settings to the values for your setup.  
If not using the settings the config file will deploy to default.

Chart will deploy with minimal settings but all parents in the config section for chirpstack and gb needs to exists.

Use the region section in chirpstack to select the regions you want to enable. The frequency settings will be derived from preconfigured json files.
For a complete list of avalible regions check configurations folder or visit [chirpstack](https://www.chirpstack.io/) where you also find a description of all settings.


```yaml
_common:
  namespace: "chirpstack"
  mqtt:
    host: broker.local
    port: 1883
    username: user
    password: pass
    qos: 2
    cleanSession: false
    clientId: "chirpstack"
    keepAliveInterval: "30s" # This defines the maximum time that that should pass without communication between the client and server.

  postgres: 
    settings: # common settings for postgres
      postgresUser: chirpstack
      adminPassword: chirpstack
      dbName: chirpstack
      role: chirpstack
      password: chirpstack
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
      enabled: true # if enabled, host and port needs to point to external postgres. If not enabled local postgres will be deployed and used
      host: "gres.local"
      port: 1234
  redis: 
    settings:
      tls: false
      cluster : false 
    external:
      enabled: false # if enabled, host and port needs to point to external redis. If not enabled local redis will be deployed and used
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
  region:
    - name: "eu868" # name on region to enable for the chart to select the correct settings file
      settings: 
        gateway:
          private : false  # If enabled, gateways can only be used by devices under the same tenant.
          enabledBackend: "mqtt" # The enabled backend type.
          mqtt: 
            server: "tcp://127.0.0.1:1883"
            username: ""
            password: ""
            qos: 0
            cleanSession: true
            clientId: ""
            keepAliveInterval: "30s"
            caCert: ""
            tlsCert: ""
            tlsKey: ""
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
    #- name: "eu433"
  useCommonMqttSettings: true
  config: 
    logging: 
      level: "idle"   # Options are: trace, debug, info, warn error.
      extra: |
        # extra settings not included in helm chart
    postgresql:
      extra: |
        # extra settings for postgresql not included from the common
    redis: 
      extra: |
        # extra settings for redis not included from the common
    api: 
      interface: "0.0.0.0"
      port: "8080"   # port to bind the API interface to.
      secret: "iLikePink" # This secret is used for generating login and API tokens, make sure this
      # is never exposed. Changing this secret will invalidate all login and API
      # tokens. The following command can be used to generate a random secret:
      #   openssl rand -base64 32
      extra: |
        # extra settings for api not included in helm chart
    gateway: 
        # CA certificate and key file (optional).
        # If setting the CA certificate and key file options, ChirpStack 
        # will generate client certificates which can be used by the gateway for
        # authentication and authorization. The Common Name of the certificate will
        # be set to the Gateway ID.
        #
        # The ca_key is expected to be in PKCS#8 format (you can use openssl to
        # convert to PKCS#8).
      caCert: ""
      caKey: ""
      clientCertLiftime: "11months 30days 3h 50m 24s" # This defines how long (after generating) the certificate remains valid.
      allowUnknownGateways: false  # If set to true, then uplinks received from gateways not configured in ChirpStack will be allowed.
      extra: |
        # extra settings not included in helm chart
    network: # all network settings except regions which is set in chirpstack.region
      netId: "000000" # Network identifier (NetID, 3 bytes) encoded as HEX (e.g. 010203).
      secondaryNetIds: []
      devAddrPrefixes: []
      deduplicationDelay: "200ms"
      macCommandsDisabled: false
      adrPlugins: []
      scheduler: 
        interval: "1s"
        classALockDuration: "5s"
        classCLockDuration: "5s"
        multicastClassCMargin: "5s"
        multicastClassBMargin: "5s"
        extra: |
        # extra settings for network scheduler not included in helm chart
      extra: |
        # extra settings for network not included in helm chirpstack-configuration
    monitoring: 
      bind: ""
      backendInterfacesLogMaxHistory: 10
      metaLogMaxHistory: 10
      gatewayFrameLogMaxHistory: 10
      deviceFrameLogMaxHistory: 10
      deviceEventLogMaxHistory: 10
      perGatewayFrameLogMaxHistory: 10
      perGatewayFrameLogTtl: "1month 13h 26m 24s"
      perDeviceFrameLogMaxHistory: 10
      perDeviceFrameLogTtl: "1month 13h 26m 24s"
      perDeviceEventLogMaxHistory: 10
      perDeviceEventLogTtl: "1month 13h 26m 24s"
      extra: |
        # extra settings for api not included in helm chart
    integration:
      enabled: # integrations to enable for chirpstack
        - mqtt
      mqtt: 
        eventTopic: "application/{{application_id}}/device/{{dev_eui}}/event/{{event}}" # Event topic template.
        commandTopic: "application/{{application_id}}/device/{{dev_eui}}/command/{{command}}" # Command topic template. 
        json: true # Use JSON encoding instead of Protobuf (binary).
        server: "tcp://127.0.0.1:1883/"
        username: ""
        password: ""
        qos: 2
        cleanSession: true
        clientId: "chirpstack"
        keepAliveInterval: "30s"
        caCert: ""
        tlsCert: ""
        tlsKey: ""
        client: 
          caCert: ""
          caKey: ""
          clientCertLiftime: "11months 30days 3h 50m 24s"
          extra: |
            # extra settings for mqtt client integration not included in helm chart
        extra: |
          # extra settings for mqtt integration not included in helm chart
      postgresql:
        useCustom: false # overrides the common settings for postgresql for the integration
        dsn: "postgresql://chirpstack_integration:chirpstack_integration@localhost/chirpstack_integration?sslmode=disable"
        maxOpenConnections: 10
        caCert: ""
        extra: |
          # extra settings for postgresql integration not included in helm chart
      amqp:
        url: "amqp://guest:guest@localhost:5672"
        eventRoutingKey: "application.{{application_id}}.device.{{dev_eui}}.event.{{event}}"
        json: true
        extra: |
          # extra settings for amqp integration not included in helm chart
      kafka:
        brokers:
          - "localhost:9092"
        tls: false
        topic: "chirpstack"
        eventKey: "application.{{application_id}}.device.{{dev_eui}}.event.{{event}}"
        username: ""
        password: ""
        mechanism: "PLAIN"
        json: true
        extra: |
          # extra settings for kafka integration not included in helm chart
    codec:
      codecJs:
        maxExecutionTime: "100ms"
        extra: |
          # extra settings for codecJS not included in helm chart
      extra: |
          # extra settings for codec not included in helm chart
    userAuthentication:
      enabled: "openid_connect" # or "internal" or "oauth2"
      openidConnect:
        registrationEnabled: true
        registrationCallbackUrl: "https://example.com/callback"
        providerUrl: "https://auth.example.com"
        clientId: "example-client-id"
        clientSecret: "example-client-secret"
        redirectUrl: "https://example.com/auth/oidc/callback"
        logoutUrl: "https://example.com/logout"
        loginRedirect: true
        loginLabel: "Login with OpenID"
        assumeEmailVerified: true
        scopes:
          - email
          - profile
        extra: |
          # extra settings for userAuthentication openidConnect not included in helm chart
      oauth2:
        provider: "clerk" # or another supported provider
        registrationEnabled: false
        registrationCallbackUrl: "https://example.com/oauth2-callback"
        clientId: "oauth2-client-id"
        clientSecret: "oauth2-client-secret"
        authUrl: "https://auth.example.com/oauth2/auth"
        tokenUrl: "https://auth.example.com/oauth2/token"
        userinfoUrl: "https://auth.example.com/oauth2/userinfo"
        redirectUrl: "https://example.com/auth/oauth2/callback"
        logoutUrl: "https://example.com/logout"
        loginRedirect: false
        loginLabel: "Login with OAuth2"
        assumeEmailVerified: false
        scopes:
          - email
        extra: |
          # extra settings for userAuthentication oauth2 not included in helm chart
      extra: |
          # extra settings for userAuthentication not included in helm chart
    joinServer:
      extra: |
        # extra settings for joinServer not included in helm chart
    backendInterfaces:
      bind: ""
      caCert: ""
      tlsCert: ""
      tlsKey: ""
      extra: |
        # extra settings for backendInterfaces not included in helm chart
    roaming: 
      resolveNetIdDomainSuffix: ""
      default: 
        enabled: false
        asyncTimeout: "0s"
        passiveRoamingLifetime: "0s"
        passiveRoamingKekLabel: ""
        passiveRoamingValidateMic: false
        server: ""
        useTargetRoleSuffix: false
        caCert: ""
        tlsCert: ""
        tlsKey: ""
        authorizationHeader: ""
        extra: |
          # extra settings for roaming default not included in helm chart
      extra: |
        # extra settings for roaming not included in helm chart
    ui:
      tileserver: "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
      mapAttribution: "&copy; <a href=\"http://osm.org/copyright\">OpenStreetMap</a> contributors"
      extra: |
        # extra settings for ui not included in helm chart
    extra: |
      # extra config fields not included in the helm chart.

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
      name: chirpstack-gateway-bridge-configuration
  service:
    port: 1700
    type: ClusterIP
  useCommonMqttSettings: true # mqtt values are populated in the config automatically from common mqtt settings, otherwise it needs to be specified in the config section.
  config:
    general:
      logLevel: 4
      logJson: false
      logToSyslog: false
      extra: | 
        # extra settings for general not included in chart
    filters:
      netIds: []
      joinEuis: []
      extra: | 
        # extra settings for filters not included in chart
    backend: 
      type: "semtech_udp"
      semtechUdp:
        udpBind: "0.0.0.0:1700"
        fakeRxTime: false
        extra: | 
          # extra settings for backend semtechUdp not included in chart
      concentratord:
        eventUrl: "ipc:///tmp/concentratord_event"
        commandUrl: "ipc:///tmp/concentratord_command"
        extra: | 
          # extra settings for backend concetratord not included in chart
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
        extra: | 
          # extra settings for backend basicstation not included in chart
      extra: | 
        # extra settings for backend not included in chart
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
            extra: | 
              # extra settings for integration mqtt auth generic not included in chart
          gcpCloudIotCore:
            server: "ssl://mqtt.googleapis.com:8883"
            deviceId: ""
            projectId: ""
            cloudRegion: ""
            registryId: ""
            jwtExpiration: "24h0m0s"
            jwtKeyFile: ""
            extra: | 
              # extra settings for integration mqtt auth gcpCloudIotCore not included in chart
          azureIotHub:
            deviceConnectionString: ""
            sasTokenExpiration: "24h0m0s"
            deviceId: ""
            hostname: ""
            tlsCert: ""
            tlsKey: ""
            extra: | 
              # extra settings for integration mqtt auth azureIotHub not included in chart
          extra: | 
            # extra settings for integration mqtt auth not included in chart
        extra: | 
            # extra settings for integration mqtt not included in chart
      extra: | 
            # extra settings for integration not included in chart
    metrics:
      prometheus:
        endpointEnabled: false
        bind: ""
        extra: | 
          # extra settings for metrics prometheus not included in chart
      extra: | 
        # extra settings for metrics not included in chart
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
        extra: | 
          # extra settings for metaData dynamic not included in chart
      extra: | 
        # extra settings for metaData not included in chart
    commands: |
      # Example:
      # [commands.commands.reboot]
      # max_execution_duration="1s"
      # command="/usr/bin/reboot"
    extra: | 
      # extra settings for config not included in chart


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