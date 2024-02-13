# Nginx configuration for Opentelemetry Collector in Api Discovery

The Apiconnect opentelemetry collector supports when the nginx is configured in any of the below ways

**Pre-req:** Ensure nginx is installed 

### 1. Using otel-webserver-module
https://github.com/open-telemetry/opentelemetry-cpp-contrib/tree/main/instrumentation/otel-webserver-module

These webserver modules can be found as downloadable in [releases](https://github.com/open-telemetry/opentelemetry-cpp-contrib/releases)

Update the nginx deployment to use the OTEL_EXPORTER_OTLP_ENDPOINT to point the Apiconnect discovery collector
```
        env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: management-api-discovery-otel-collector.{namespace}.svc:5555
```

These [Webserver module](https://github.com/open-telemetry/opentelemetry-cpp-contrib/tree/main/instrumentation/otel-webserver-module#configuration-1) configurations in opentelemetry config can be used to explitly give some trace parameters value.

And the datasource name of the APIs collected through this will be named after the attribute NginxModuleServiceName in opentelemetry conf.

Example for configuring nginx config and opentelemetry config using this module in nginx

**Sample nginx-config**

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
  namespace: nginx
data:
  nginx-cm.yaml: |
    load_module /opt/opentelemetry-webserver-sdk/WebServerModule/Nginx/1.25.3/ngx_http_opentelemetry_module.so;
    events {}
    http {
      server_tokens off;
      client_max_body_size 32m;
      include /opt/opentelemetry_module.conf;
      server {
        listen 80;
        location / {
          # The following statement will proxy traffic to the upstream named Backend
          proxy_pass http://{backend-service}:{port}/;
        }
      }
    }
```

**Sample Opentelemetry config**

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: opentelemetry-conf
  namespace: nginx
data:
  opentelemetry_module.conf: |
    NginxModuleEnabled ON;
    NginxModuleOtelSpanExporter otlp;
    NginxModuleOtelExporterEndpoint management-api-discovery-otel-collector.istio-system.svc:5555;
    NginxModuleServiceName DemoService;
    NginxModuleServiceNamespace DemoServiceNamespace;
    NginxModuleServiceInstanceId DemoInstanceId;
    NginxModuleResolveBackends ON;
    NginxModuleTraceAsError ON;
```
**Sample volumes and mounts in nginx deployment to use the above configs**

```
        volumeMounts:
        - mountPath: /opt/opentelemetry_module.conf
          subPath: opentelemetry_module.conf
          name: opentelemetry-conf
        - mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
          name: nginx-conf			  
      volumes:
      - name: nginx-conf
        configMap:
          name: nginx-conf
          items:
            - key: nginx-cm.yaml
              path: nginx.conf
      - name: opentelemetry-conf
        configMap:
          name: opentelemetry-conf
          items:
            - key: opentelemetry_module.conf
              path: opentelemetry_module.conf	
```



### 2. Using nginx-instrumentation 
from https://github.com/open-telemetry/opentelemetry-cpp-contrib/tree/main/instrumentation/nginx

Similar to the above module, the nginx can be configured to point the Apiconnect discovery collector using
```
        env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: management-api-discovery-otel-collector.{namespace}.svc:5555
```
Or can be configured in otel-nginx.toml as in the opentelemetry-cpp-contrib. 

The module helps configuring the nginx.conf to use different variables to customize the traces that sends to Discovery service. [nginx-directives](https://github.com/open-telemetry/opentelemetry-cpp-contrib/tree/main/instrumentation/nginx#nginx-directives)

Example for configuring nginx config and opentelemetry config using this module in nginx

**Sample nginx-config**

```
apiVersion: v1
kind: ConfigMap
metadata:
 name: nginx-conf
 namespace: nginx
data:
 nginx-cm.yaml: |
   load_module /usr/lib/nginx/modules/otel_ngx_module.so;
   events {}
   http {
     opentelemetry_config /conf/otel-nginx.toml;
     server {
       listen 80;
       location = / {
         # The following statement will proxy traffic to the upstream named Backend
         proxy_pass http://hello:8080/;
       }
     }
   }
```

**Sample volumes and mounts in nginx deployment to use the above config**

```
apiVersion: v1
kind: ConfigMap
metadata:
 name: nginx-conf
 namespace: nginx
data:
 nginx-cm.yaml: |
   load_module /usr/lib/nginx/modules/otel_ngx_module.so;
   events {}
   http {
     opentelemetry_config /conf/otel-nginx.toml;
     server {
       listen 80;
       location = / {
         # The following statement will proxy traffic to the upstream named Backend
         proxy_pass http://{backend-service}:{port}/;
       }
     }
   }
```

### 3. Using Ingress-nginx controller
The configMap need to be configured with enable-opentelemetry (true) and otlp-collector-host to collect the traces from the backend service
https://kubernetes.github.io/ingress-nginx/user-guide/third-party-addons/opentelemetry/

