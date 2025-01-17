# Migration Guide: From v1 to v2

How to Migrate from Traefik v1 to Traefik v2.
{: .subtitle }

The version 2 of Traefik introduces a number of breaking changes,
which require one to update their configuration when they migrate from v1 to v2.
The goal of this page is to recapitulate all of these changes, and in particular to give examples,
feature by feature, of how the configuration looked like in v1, and how it now looks like in v2.

!!! info "Migration Helper"

    We created a tool to help during the migration: [traefik-migration-tool](https://github.com/containous/traefik-migration-tool)

    This tool allows to:

    - convert `Ingress` to Traefik `IngressRoute` resources.
    - convert `acme.json` file from v1 to v2 format.
    - migrate the static configuration contained in the file `traefik.toml` to a Traefik v2 file.

## Frontends and Backends Are Dead... <br/>... Long Live Routers, Middlewares, and Services

During the transition from v1 to v2, a number of internal pieces and components of Traefik were rewritten and reorganized.
As such, the combination of core notions such as frontends and backends has been replaced with the combination of [routers](../routing/routers/index.md), [services](../routing/services/index.md), and [middlewares](../middlewares/overview.md).

Typically, a router replaces a frontend, and a service assumes the role of a backend, with each router referring to a service.
However, even though a backend was in charge of applying any desired modification on the fly to the incoming request,
the router defers that responsibility to another component.
Instead, a dedicated middleware is now defined for each kind of such modification.
Then any router can refer to an instance of the wanted middleware.

!!! example "One frontend with basic auth and one backend, become one router, one service, and one basic auth middleware."

    !!! info "v1"

    ```yaml tab="Docker"
    labels:
      - "traefik.frontend.rule=Host:test.localhost;PathPrefix:/test"
      - "traefik.frontend.auth.basic.users=test:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/,test2:$$apr1$$d9hr9HBB$$4HxwgUir3HP4EsggP/QNo0"
    ```

    ```yaml tab="K8s Ingress"
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: traefik
      namespace: kube-system
      annotations:
        kubernetes.io/ingress.class: traefik
        traefik.ingress.kubernetes.io/rule-type: PathPrefix
    spec:
      rules:
      - host: test.locahost
        http:
          paths:
          - path: /test
            backend:
              serviceName: server0
              servicePort: 80
          - path: /test
            backend:
              serviceName: server1
              servicePort: 80
    ```

    ```toml tab="File (TOML)"
    [frontends]
      [frontends.frontend1]
        entryPoints = ["http"]
        backend = "backend1"
    
        [frontends.frontend1.routes]
          [frontends.frontend1.routes.route0]
            rule = "Host:test.localhost"
          [frontends.frontend1.routes.route0]
            rule = "PathPrefix:/test"
        
        [frontends.frontend1.auth]
          [frontends.frontend1.auth.basic]
            users = [
              "test:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/",
              "test2:$apr1$d9hr9HBB$4HxwgUir3HP4EsggP/QNo0",
            ]
    
    [backends]
      [backends.backend1]
        [backends.backend1.servers.server0]
          url = "http://10.10.10.1:80"
        [backends.backend1.servers.server1]
          url = "http://10.10.10.2:80"
    
        [backends.backend1.loadBalancer]
          method = "wrr"
    ```

    !!! info "v2"

    ```yaml tab="Docker"
    labels:
      - "traefik.http.routers.router0.rule=Host(`bar.com`) && PathPrefix(`/test`)"
      - "traefik.http.routers.router0.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=test:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/,test2:$$apr1$$d9hr9HBB$$4HxwgUir3HP4EsggP/QNo0"
    ```

    ```yaml tab="K8s IngressRoute"
    # The definitions below require the definitions for the Middleware and IngressRoute kinds.
    # https://docs.traefik.io/v2.0/providers/kubernetes-crd/#traefik-ingressroute-definition
    apiVersion: traefik.containo.us/v1alpha1
    kind: Middleware
    metadata:
      name: basicauth
      namespace: foo
    
    spec:
      basicAuth:
        users:
          - test:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/
          - test2:$apr1$d9hr9HBB$4HxwgUir3HP4EsggP/QNo0
    
    ---
    apiVersion: traefik.containo.us/v1alpha1
    kind: IngressRoute
    metadata:
      name: ingressroutebar
    
    spec:
      entryPoints:
        - http
      routes:
      - match: Host(`test.localhost`) && PathPrefix(`/test`)
        kind: Rule
        services:
        - name: server0
          port: 80
        - name: server1
          port: 80
        middlewares:
        - name: basicauth
          namespace: foo
    ```

    ```toml tab="File (TOML)"
    [http.routers]
      [http.routers.router0]
        rule = "Host(`test.localhost`) && PathPrefix(`/test`)"
        middlewares = ["auth"]
        service = "my-service"
    
    [http.services]
      [[http.services.my-service.loadBalancer.servers]]
        url = "http://10.10.10.1:80"
      [[http.services.my-service.loadBalancer.servers]]
        url = "http://10.10.10.2:80"
    
    [http.middlewares]
      [http.middlewares.auth.basicAuth]
        users = [
          "test:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/",
          "test2:$apr1$d9hr9HBB$4HxwgUir3HP4EsggP/QNo0",
        ]
    ```

    ```yaml tab="File (YAML)"
    http:
      routers:
        router0:
          rule: "Host(`test.localhost`) && PathPrefix(`/test`)"
          service: my-service
          middlewares:
            - auth
    
      services:
        my-service:
          loadBalancer:
            servers:
              - url: http://10.10.10.1:80
              - url: http://10.10.10.2:80
    
      middlewares:
        auth:
          basicAuth:
            users:
              - "test:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/"
              - "test2:$apr1$d9hr9HBB$4HxwgUir3HP4EsggP/QNo0"
    ```

## TLS Configuration Is Now Dynamic, per Router.

TLS parameters used to be specified in the static configuration, as an entryPoint field.
With Traefik v2, a new dynamic TLS section at the root contains all the desired TLS configurations.
Then, a [router's TLS field](../routing/routers/index.md#tls) can refer to one of the [TLS configurations](../https/tls.md) defined at the root, hence defining the [TLS configuration](../https/tls.md) for that router.

!!! example "TLS on web-secure entryPoint becomes TLS option on Router-1"

    !!! info "v1"
    
    ```toml tab="File (TOML)"
    # static configuration
    [entryPoints]
      [entryPoints.web-secure]
        address = ":443"
    
        [entryPoints.web-secure.tls]
          minVersion = "VersionTLS12"
          cipherSuites = [
            "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256",
            "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384",
            "TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305",
            "TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305",
            "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256",
            "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256",
           ]
          [[entryPoints.web-secure.tls.certificates]]
            certFile = "path/to/my.cert"
            keyFile = "path/to/my.key"
    ```

    ```bash tab="CLI"
    --entryPoints='Name:web-secure Address::443 TLS:path/to/my.cert,path/to/my.key TLS.MinVersion:VersionTLS12 TLS.CipherSuites:TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256'
    ```

    !!! info "v2"

    ```toml tab="File (TOML)"
    # dynamic configuration
    [http.routers]
      [http.routers.Router-1]
        rule = "Host(`bar.com`)"
        service = "service-id"
        # will terminate the TLS request
        [http.routers.Router-1.tls]
          options = "myTLSOptions"
    
    [[tls.certificates]]
      certFile = "/path/to/domain.cert"
      keyFile = "/path/to/domain.key"
    
    [tls.options]
      [tls.options.default]
        minVersion = "VersionTLS12"
    
      [tls.options.myTLSOptions]
        minVersion = "VersionTLS13"
        cipherSuites = [
            "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256",
            "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384",
            "TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305",
            "TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305",
            "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256",
            "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256",
            ]
    ```

    ```yaml tab="File (YAML)"
    http:
      routers:
        Router-1:
          rule: "Host(`bar.com`)"
          service: service-id
          # will terminate the TLS request
          tls:
            options: myTLSOptions
    
    tls:
      certificates:
        - certFile: /path/to/domain.cert
          keyFile: /path/to/domain.key
      options:
        myTLSOptions:
          minVersion: VersionTLS13
          cipherSuites:
	        - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
	        - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
	        - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
	        - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
	        - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
    ```

    ```yaml tab="K8s IngressRoute"
    # The definitions below require the definitions for the TLSOption and IngressRoute kinds.
    # https://docs.traefik.io/v2.0/providers/kubernetes-crd/#traefik-ingressroute-definition
    apiVersion: traefik.containo.us/v1alpha1
    kind: TLSOption
    metadata:
      name: mytlsoption
      namespace: default
    
    spec:
      minVersion: VersionTLS13
      cipherSuites:
	    - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
	    - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
	    - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
	    - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
	    - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
    
    ---
    apiVersion: traefik.containo.us/v1alpha1
    kind: IngressRoute
    metadata:
      name: ingressroutebar
    
    spec:
      entryPoints:
        - web
      routes:
        - match: Host(`bar.com`)
          kind: Rule
          services:
            - name: whoami
              port: 80
      tls:
        options:
          name: mytlsoption
          namespace: default
    ```

    ```yaml tab="Docker"
    labels:
      # myTLSOptions must be defined by another provider, in this instance in the File Provider.
      # see the cross provider section
      - "traefik.http.routers.router0.tls.options=myTLSOptions@file"
    ```

## HTTP to HTTPS Redirection Is Now Configured on Routers

Previously on Traefik v1, the redirection was applied on an entry point or on a frontend.
With Traefik v2 it is applied on a [Router](../routing/routers/index.md). 

To apply a redirection, one of the redirect middlewares, [RedirectRegex](../middlewares/redirectregex.md) or [RedirectScheme](../middlewares/redirectscheme.md), has to be configured and added to the router middlewares list.

!!! example "HTTP to HTTPS redirection"

    !!! info "v1"
    
    ```toml tab="File (TOML)"
    # static configuration
    defaultEntryPoints = ["http", "https"]
    
    [entryPoints]
      [entryPoints.http]
        address = ":80"
        [entryPoints.http.redirect]
          entryPoint = "https"

      [entryPoints.https]
        address = ":443"
        [entryPoints.https.tls]
          [[entryPoints.https.tls.certificates]]
            certFile = "examples/traefik.crt"
            keyFile = "examples/traefik.key"
    ```

    ```bash tab="CLI"
    --entrypoints=Name:web Address::80 Redirect.EntryPoint:web-secure
    --entryPoints='Name:web-secure Address::443 TLS:path/to/my.cert,path/to/my.key'
    ```

    !!! info "v2"

    ```yaml tab="Docker"
    labels:
    - traefik.http.routers.web.rule=Host(`foo.com`)
    - traefik.http.routers.web.entrypoints=web
    - traefik.http.routers.web.middlewares=redirect@file
    - traefik.http.routers.web-secured.rule=Host(`foo.com`)
    - traefik.http.routers.web-secured.entrypoints=web-secure
    - traefik.http.routers.web-secured.tls=true
    ```

    ```yaml tab="K8s IngressRoute"
    apiVersion: traefik.containo.us/v1alpha1
    kind: IngressRoute
    metadata:
      name: http-redirect-ingressRoute
    
    spec:
      entryPoints:
        - web
      routes:
        - match: Host(`foo.com`)
          kind: Rule
          services:
            - name: whoami
              port: 80
          middlewares:
            - name: redirect
    
    ---
    apiVersion: traefik.containo.us/v1alpha1
    kind: IngressRoute
    metadata:
      name: https-ingressRoute
    
    spec:
      entryPoints:
        - web-secure
      routes:
        - match: Host(`foo`)
          kind: Rule
          services:
            - name: whoami
              port: 80
      tls: {}
      
    ---
    apiVersion: traefik.containo.us/v1alpha1
    kind: Middleware
    metadata:
      name: redirect
    spec:
      redirectScheme:
        scheme: https
      
    ```

    ```toml tab="File (TOML)"
    ## static configuration
    # traefik.toml
    
    [entryPoints.web]
      address = ":80"
    
    [entryPoints.web-secure]
      address = ":443"
    
    ##---------------------##
    
    ## dynamic configuration
    # dynamic-conf.toml
    
    [http.routers]
      [http.routers.router0]
        rule = "Host(`foo.com`)"
        service = "my-service"
        entrypoints = ["web"]
        middlewares = ["redirect"]
    
    [http.routers.router1]
        rule = "Host(`foo.com`)"
        service = "my-service"
        entrypoints = ["web-secure"]
        [http.routers.router1.tls]
        
    [http.services]
      [[http.services.my-service.loadBalancer.servers]]
        url = "http://10.10.10.1:80"
      [[http.services.my-service.loadBalancer.servers]]
        url = "http://10.10.10.2:80"
    
    [http.middlewares]
      [http.middlewares.redirect.redirectScheme]
        scheme = "https"
    
    [[tls.certificates]]
      certFile = "/path/to/domain.cert"
      keyFile = "/path/to/domain.key"
    ```

    ```yaml tab="File (YAML)"
    ## static configuration
    # traefik.yml
    
    entryPoints:
      web:
        address: ":80"
    
      web-secure:
        address: ":443"
    
    ##---------------------##
    
    ## dynamic configuration
    # dynamic-conf.yml
    
    http:
      routers:
        router0:
          rule: "Host(`foo.com`)"
          entryPoints:
            - web
          middlewares:
            - redirect
          service: my-service
    
        router1:
          rule: "Host(`foo.com`)"
          entryPoints:
            - web-secure
          service: my-service
          tls: {}
    
      services:
        my-service:
          loadBalancer:
            servers:
              - url: http://10.10.10.1:80
              - url: http://10.10.10.2:80
    
      middlewares:
        redirect:
          redirectScheme:
            scheme: https
    
    tls:
      certificates:
        - certFile: /app/certs/server/server.pem
          keyFile: /app/certs/server/server.pem
    ```

## Strip and Rewrite Path Prefixes

With the new core notions of v2 (introduced earlier in the section
["Frontends and Backends Are Dead... Long Live Routers, Middlewares, and Services"](#frontends-and-backends-are-dead-long-live-routers-middlewares-and-services)),
transforming the URL path prefix of incoming requests is configured with [middlewares](../../middlewares/overview/),
after the routing step with [router rule `PathPrefix`](https://docs.traefik.io/v2.0/routing/routers/#rule).

Use Case: Incoming requests to `http://company.org/admin` are forwarded to the webapplication "admin",
with the path `/admin` stripped, e.g. to `http://<IP>:<port>/`. In this case, you must:

* First, configure a router named `admin` with a rule matching at least the path prefix with the `PathPrefix` keyword,
* Then, define a middlware of type [`stripprefix`](../../middlewares/stripprefix/), which remove the prefix `/admin`, associated to the router `admin`.

!!! example "Strip Path Prefix When Forwarding to Backend"

    !!! info "v1"

    ```yaml tab="Docker"
    labels:
      - "traefik.frontend.rule=Host:company.org;PathPrefixStrip:/admin"
    ```

    ```yaml tab="Kubernetes Ingress"
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: traefik
      annotations:
        kubernetes.io/ingress.class: traefik
        traefik.ingress.kubernetes.io/rule-type: PathPrefixStrip
    spec:
      rules:
      - host: company.org
        http:
          paths:
          - path: /admin
            backend:
              serviceName: admin-svc
              servicePort: admin
    ```

    ```toml tab="File (TOML)"
    [frontends.admin]
      [frontends.admin.routes.admin_1]
      rule = "Host:company.org;PathPrefixStrip:/admin"
    ```

    !!! info "v2"

    ```yaml tab="Docker"
    labels:
      - "traefik.http.routers.admin.rule=Host(`company.org`) && PathPrefix(`/admin`)"
      - "traefik.http.middlewares.admin-stripprefix.stripprefix.prefixes=/admin"
      - "traefik.http.routers.web.middlewares=admin-stripprefix@docker"
    ```

    ```yaml tab="Kubernetes IngressRoute"
    ---
    apiVersion: traefik.containo.us/v1alpha1
    kind: IngressRoute
    metadata:
      name: http-redirect-ingressRoute
      namespace: admin-web
    spec:
      entryPoints:
        - web
      routes:
        - match: Host(`company.org`) && PathPrefix(`/admin`)
          kind: Rule
          services:
            - name: admin-svc
              port: admin
          middlewares:
            - name: admin-stripprefix
    ---
    kind: Middleware
    metadata:
      name: admin-stripprefix
    spec:
      stripPrefix:
        prefixes:
          - /admin
    ```

    ```toml tab="File (TOML)"
    ## Dynamic configuration
    # dynamic-conf.toml

    [http.routers.router1]
        rule = "Host(`company.org`) && PathPrefix(`/admin`)"
        service = "admin-svc"
        entrypoints = ["web"]
        middlewares = ["admin-stripprefix"]

    [http.middlewares]
      [http.middlewares.admin-stripprefix.stripPrefix]
      prefixes = ["/admin"]
    
    # ...
    ```

    ```yaml tab="File (YAML)"
    ## Dynamic Configuration
    # dynamic-conf.yml

    # As YAML Configuration File
    http:
      routers:
        admin:
          service: admin-svc
          middlewares:
            - "admin-stripprefix"
          rule: "Host(`company.org`) && PathPrefix(`/admin`)"

      middlewares:
        admin-stripprefix:
          addPrefix:
            prefix: "/admin"

    # ...
    ```

??? question "What About Other Path Transformations?"

    Instead of removing the path prefix with the [`stripprefix` middleware](../../middlewares/stripprefix/), you can also:

    * Add a path prefix with the [`addprefix` middleware](../../middlewares/addprefix/)
    * Replace the complete path of the request with the [`replacepath` middleware](../../middlewares/replacepath/)
    * ReplaceRewrite path using Regexp with the [`replacepathregex` middleware](../../middlewares/replacepathregex/)
    * And a lot more on the [`middlewares` page](../../middlewares/overview/)

## ACME (LetsEncrypt)

[ACME](../https/acme.md) is now a certificate resolver (under a certificatesResolvers section) but remains in the static configuration.

!!! example "ACME from provider to a specific Certificate Resolver"

    !!! info "v1"
    
    ```toml tab="File (TOML)"
    # static configuration
    defaultEntryPoints = ["web-secure","web"]
    
    [entryPoints.web]
    address = ":80"
      [entryPoints.web.redirect]
      entryPoint = "webs"
    [entryPoints.web-secure]
      address = ":443"
      [entryPoints.https.tls]
    
    [acme]
      email = "your-email-here@my-awesome-app.org"
      storage = "acme.json"
      entryPoint = "web-secure"
      onHostRule = true
      [acme.httpChallenge]
        entryPoint = "web"
    ```

    ```bash tab="CLI"
    --defaultentrypoints=web-secure,web
    --entryPoints=Name:web Address::80 Redirect.EntryPoint:web-secure
    --entryPoints=Name:web-secure Address::443 TLS
    --acme.email=your-email-here@my-awesome-app.org
    --acme.storage=acme.json
    --acme.entryPoint=web-secure
    --acme.onHostRule=true
    --acme.httpchallenge.entrypoint=http
    ```

    !!! info "v2"

    ```toml tab="File (TOML)"
    # static configuration
    [entryPoints]
      [entryPoints.web]
        address = ":80"
    
      [entryPoints.web-secure]
        address = ":443"
    
    [certificatesResolvers.sample.acme]
      email = "your-email@your-domain.org"
      storage = "acme.json"
      [certificatesResolvers.sample.acme.httpChallenge]
        # used during the challenge
        entryPoint = "web"
    ```

    ```yaml tab="File (YAML)"
    entryPoints:
      web:
        address: ":80"
    
      web-secure:
        address: ":443"
    
    certificatesResolvers:
      sample:
        acme:
          email: your-email@your-domain.org
          storage: acme.json
          httpChallenge:
            # used during the challenge
            entryPoint: web
    ```

    ```bash tab="CLI"
    --entryPoints.web.address=":80"
    --entryPoints.websecure.address=":443"
    --certificatesResolvers.sample.acme.email: your-email@your-domain.org
    --certificatesResolvers.sample.acme.storage: acme.json
    --certificatesResolvers.sample.acme.httpChallenge.entryPoint: web
    ```

## Traefik Logs

In the v2, all the [log configuration](../observability/logs.md) remains in the static part but are unified under a `log` section.
There is no more log configuration at the root level.

!!! example "Simple log configuration"

    !!! info "v1"

    ```toml tab="File (TOML)"
    # static configuration
    logLevel = "DEBUG"
    
    [traefikLog]
      filePath = "/path/to/traefik.log"
      format   = "json"
    ```

    ```bash tab="CLI"
    --logLevel="DEBUG"
    --traefikLog.filePath="/path/to/traefik.log"
    --traefikLog.format="json"
    ```

    !!! info "v2"

    ```toml tab="File (TOML)"
    # static configuration
    [log]
      level = "DEBUG"
      filePath = "/path/to/log-file.log"
      format = "json"
    ```

    ```yaml tab="File (YAML)"
    # static configuration
    log:
      level: DEBUG
      filePath: /path/to/log-file.log
      format: json
    ```

    ```bash tab="CLI"
    --log.level="DEBUG"
    --log.filePath="/path/to/traefik.log"
    --log.format="json"
    ```

## Tracing

Traefik v2 retains OpenTracing support. The `backend` root option from the v1 is gone, you just have to set your [tracing configuration](../observability/tracing/overview.md).

!!! example "Simple Jaeger tracing configuration"

    !!! info "v1"

    ```toml tab="File (TOML)"
    # static configuration
    [tracing]
      backend = "jaeger"
      servicename = "tracing"
      [tracing.jaeger]
        samplingParam = 1.0
        samplingServerURL = "http://12.0.0.1:5778/sampling"
        samplingType = "const"
        localAgentHostPort = "12.0.0.1:6831"
    ```

    ```bash tab="CLI"
    --tracing.backend="jaeger"
    --tracing.servicename="tracing"
    --tracing.jaeger.localagenthostport="12.0.0.1:6831"
    --tracing.jaeger.samplingparam="1.0"
    --tracing.jaeger.samplingserverurl="http://12.0.0.1:5778/sampling"
    --tracing.jaeger.samplingtype="const"
    ```

    !!! info "v2"

    ```toml tab="File (TOML)"
    # static configuration
    [tracing]
      servicename = "tracing"
      [tracing.jaeger]
        samplingParam = 1.0
        samplingServerURL = "http://12.0.0.1:5778/sampling"
        samplingType = "const"
        localAgentHostPort = "12.0.0.1:6831"
    ```

    ```yaml tab="File (YAML)"
    # static configuration
    tracing:
      servicename: tracing
      jaeger:
        samplingParam: 1
        samplingServerURL: 'http://12.0.0.1:5778/sampling'
        samplingType: const
        localAgentHostPort: '12.0.0.1:6831'
    ```

    ```bash tab="CLI"
    --tracing.servicename="tracing"
    --tracing.jaeger.localagenthostport="12.0.0.1:6831"
    --tracing.jaeger.samplingparam="1.0"
    --tracing.jaeger.samplingserverurl="http://12.0.0.1:5778/sampling"
    --tracing.jaeger.samplingtype="const"
    ```

## Metrics

The v2 retains metrics tools and allows metrics to be configured for the entrypoints and/or services.
For a basic configuration, the [metrics configuration](../observability/metrics/overview.md) remains the same.

!!! example "Simple Prometheus metrics configuration"

    !!! info "v1"

    ```toml tab="File (TOML)"
    # static configuration
    [metrics.prometheus]
      buckets = [0.1,0.3,1.2,5.0]
      entryPoint = "traefik"
    ```

    ```bash tab="CLI"
    --metrics.prometheus.buckets=[0.1,0.3,1.2,5.0]
    --metrics.prometheus.entrypoint="traefik"
    ```

    !!! info "v2"

    ```toml tab="File (TOML)"
    # static configuration
    [metrics.prometheus]
      buckets = [0.1,0.3,1.2,5.0]
      entryPoint = "metrics"
    ```

    ```yaml tab="File (YAML)"
    # static configuration
    metrics:
      prometheus:
        buckets:
          - 0.1
          - 0.3
          - 1.2
          - 5
        entryPoint: metrics
    ```

    ```bash tab="CLI"
    --metrics.prometheus.buckets=[0.1,0.3,1.2,5.0]
    --metrics.prometheus.entrypoint="metrics"
    ```

## No More Root Level Key/Values

To avoid any source of confusion, there are no more configuration at the root level.
Each root item has been moved to a related section or removed.

!!! example "From root to dedicated section"

    !!! info "v1"

    ```toml tab="File (TOML)"
    # static configuration
    checkNewVersion = false
    sendAnonymousUsage = true
    logLevel = "DEBUG"
    insecureSkipVerify = true
    rootCAs = [ "/mycert.cert" ]
    maxIdleConnsPerHost = 200
    providersThrottleDuration = "2s"
    AllowMinWeightZero = true
    debug = true
    defaultEntryPoints = ["web", "web-secure"]
    keepTrailingSlash = false
    ```

    ```bash tab="CLI"
    --checknewversion=false
    --sendanonymoususage=true
    --loglevel="DEBUG"
    --insecureskipverify=true
    --rootcas="/mycert.cert"
    --maxidleconnsperhost=200
    --providersthrottleduration="2s"
    --allowminweightzero=true
    --debug=true
    --defaultentrypoints="web","web-secure"
    --keeptrailingslash=true
    ```

    !!! info "v2"

    ```toml tab="File (TOML)"
    # static configuration
    [global]
      checkNewVersion = true
      sendAnonymousUsage = true
    
    [log]
      level = "DEBUG"
    
    [serversTransport]
      insecureSkipVerify = true
      rootCAs = [ "/mycert.cert" ]
      maxIdleConnsPerHost = 42
    
    [providers]
      providersThrottleDuration = 42
    ```

    ```yaml tab="File (YAML)"
    # static configuration
    global:
      checkNewVersion: true
      sendAnonymousUsage: true
    
    log:
      level: DEBUG
    
    serversTransport:
      insecureSkipVerify: true
      rootCAs:
        - /mycert.cert
      maxIdleConnsPerHost: 42
    
    providers:
      providersThrottleDuration: 42
    ```

    ```bash tab="CLI"
    --global.checknewversion=true
    --global.sendanonymoususage=true
    --log.level="DEBUG"
    --serverstransport.insecureskipverify=true
    --serverstransport.rootcas="/mycert.cert"
    --serverstransport.maxidleconnsperhost=42
    --providers.providersthrottleduration=42
    ```

## Dashboard

You need to activate the API to access the [dashboard](../operations/dashboard.md).
As the dashboard access is now secured by default you can either:

* define a  [specific router](../operations/api.md#configuration) with the `api@internal` service and one authentication middleware like the following example
* or use the [unsecure](../operations/api.md#insecure) option of the API

!!! info "Dashboard with k8s and dedicated router"

    As `api@internal` is not a Kubernetes service, you have to use the file provider or the `insecure` API option.

!!! example "Activate and access the dashboard"

    !!! info "v1"

    ```toml tab="File (TOML)"
    ## static configuration
    # traefik.toml
    
    [entryPoints.web-secure]
      address = ":443"
      [entryPoints.web-secure.tls]
      [entryPoints.web-secure.auth]
        [entryPoints.web-secure.auth.basic]
          users = [
            "test:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/"
          ]
    
    [api]
      entryPoint = "web-secure"
    ```

    ```bash tab="CLI"
    --entryPoints='Name:web-secure Address::443 TLS Auth.Basic.Users:test:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/'
    --api
    ```

    !!! info "v2"

    ```yaml tab="Docker"
    # dynamic configuration
    labels:
      - "traefik.http.routers.api.rule=Host(`traefik.docker.localhost`)"
      - "traefik.http.routers.api.entrypoints=web-secured"
      - "traefik.http.routers.api.service=api@internal"
      - "traefik.http.routers.api.middlewares=myAuth"
      - "traefik.http.routers.api.tls"
      - "traefik.http.middlewares.myAuth.basicauth.users=test:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/"
    ```

    ```toml tab="File (TOML)"
    ## static configuration
    # traefik.toml
    
    [entryPoints.web-secure]
      address = ":443"
    
    [api]
    
    [providers.file]
        filename = "/dynamic-conf.toml"
    
    ##---------------------##
    
    ## dynamic configuration
    # dynamic-conf.toml
    
    [http.routers.api]
      rule = "Host(`traefik.docker.localhost`)"
      entrypoints = ["web-secure"]
      service = "api@internal"
      middlewares = ["myAuth"]
      [http.routers.api.tls]
    
    [http.middlewares.myAuth.basicAuth]
      users = [
        "test:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/"
      ]
    ```

    ```yaml tab="File (YAML)"
    ## static configuration
    # traefik.yaml
    
    entryPoints:
      web-secure:
        address: ':443'
        
    api: {}
    
    providers:
      file:
        filename: /dynamic-conf.yaml
   
    ##---------------------##
    
    ## dynamic configuration
    # dynamic-conf.yaml
    
     http:
      routers:
        api:
          rule: Host(`traefik.docker.localhost`)
          entrypoints:
            - web-secure
          service: api@internal
          middlewares:
            - myAuth
          tls: {}
          
      middlewares:
        myAuth:
          basicAuth:
            users:
              - 'test:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/'
    ```

## Providers

Supported [providers](../providers/overview.md), for now:

* [ ] Azure Service Fabric
* [ ] BoltDB
* [ ] Consul
* [ ] Consul Catalog
* [x] Docker
* [ ] DynamoDB
* [ ] ECS
* [ ] Etcd
* [ ] Eureka
* [x] File
* [x] Kubernetes Ingress (without annotations)
* [x] Kubernetes IngressRoute
* [x] Marathon
* [ ] Mesos
* [x] Rancher
* [x] Rest
* [ ] Zookeeper

## Some Tips You Should Know

* Different sources of static configuration (file, CLI flags, ...) cannot be [mixed](../getting-started/configuration-overview.md#the-static-configuration).
* Now, configuration elements can be referenced between different providers by using the provider namespace notation: `@<provider>`.
  For instance, a router named `myrouter` in a File Provider can refer to a service named `myservice` defined in Docker Provider with the following notation: `myservice@docker`.
* Middlewares are applied in the same order as their declaration in router.
* If you have any questions feel free to join our [community forum](https://community.containo.us).
