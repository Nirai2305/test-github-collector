# Nginx configuration for Opentelemetry Collector in Api Discovery

**Pre-req:** Ensure nginx is installed 

The Apiconnect opentelemetry collector supports multiple different nginx opentelemetry integrations.
Some of those different integrations are outlined below

### 1. otel-webserver-module
The Otel-webserver module can be found at the following location - https://github.com/open-telemetry/opentelemetry-cpp-contrib/tree/main/instrumentation/otel-webserver-module

The webserver module can be found as downloadable in [releases](https://github.com/open-telemetry/opentelemetry-cpp-contrib/releases)

To configure the webserver module and to send traces to the Opentelemetry collector, you can update the nginx deployment to use the OTEL_EXPORTER_OTLP_ENDPOINT to point to the Apiconnect discovery collector
```
        env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: management-api-discovery-otel-collector.{namespace}.svc:5555
```

Or you can alternatively use NginxModuleOtelExporterEndpoint in opentelemetry config on your nginx server. See here for a full list of [Webserver module](https://github.com/open-telemetry/opentelemetry-cpp-contrib/tree/main/instrumentation/otel-webserver-module#configuration-1) configurations which can be set in the opentelemetry config and used to explitly give some other trace parameter values.

The datasource name of the APIs collected through this integration will be named after the attribute NginxModuleServiceName from the opentelemetry conf.

Note: This module will support a custom trace resource in future with the attribute `NginxModuleOtelResourceAttributes some.key=value;` and can be used to set datasource name via `ibm-api-discovery-datasource.name`

Below is an example of nginx config and opentelemetry config when using this module

**Sample nginx-config in a Kubernetes ConfigMap**

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

**Sample Opentelemetry config in a Kubernetes ConfigMap**

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
    NginxModuleOtelExporterEndpoint management-api-discovery-otel-collector.{namespace}.svc:5555;
    NginxModuleServiceName my-nginx-datasource-name;
    NginxModuleServiceNamespace DemoServiceNamespace;
    NginxModuleServiceInstanceId DemoInstanceId;
    NginxModuleResolveBackends ON;
    NginxModuleTraceAsError ON;
```

### 2. Opentelemetry nginx instrumentation module
The Opentelemetry nginx instrumentation module can be found at the following location -  https://github.com/open-telemetry/opentelemetry-cpp-contrib/tree/main/instrumentation/nginx

Similar to the previous module, to configure this module and to send traces to the Opentelemetry collector, you can update the nginx deployment to use the OTEL_EXPORTER_OTLP_ENDPOINT to point to the Apiconnect discovery collector
```
        env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: management-api-discovery-otel-collector.{namespace}.svc:5555
```
Or you can alternatively configure it in the otel-nginx.toml as in the opentelemetry-cpp-contrib.

This module helps configuring the nginx.conf to use different variables to customize the traces that can be sent to Discovery service. 
See here for a full list of [nginx-directives](https://github.com/open-telemetry/opentelemetry-cpp-contrib/tree/main/instrumentation/nginx#nginx-directives) configurations which can be set in the nginx config and used to explitly give some trace parameter values.
opentelemetry_attribute

The datasource name of the APIs collected through this integration will be named after the env OTEL_SERVICE_NAME from the nginx deployment or the service name from the otel-nginx.toml.

Or this module will support a custom trace attribute [opentelemetry_attribute](https://github.com/open-telemetry/opentelemetry-cpp-contrib/tree/main/instrumentation/nginx#opentelemetry_attribute) which can be used to set datasource name via `ibm-api-discovery-datasource.name`

Below is an example of nginx config when using this module

**Sample nginx-config in a Kubernetes ConfigMap**

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
### 3. Kubernetes Ingress-nginx controller

The Kubernetes Ingress-nginx controller Opentelemetry configuration can be found at the following location -  https://kubernetes.github.io/ingress-nginx/user-guide/third-party-addons/opentelemetry/ and the module used by this ingress controller is the nginx-instrumentation module. Therefore anything configurable in the instrumentation module is also configurable here in a similar way.

At a minimum, the kubernetes ingress-nginx controller ConfigMap needs to be configured with "enable-opentelemetry" (true) and "otlp-collector-host" (management-api-discovery-otel-collector.{namespace}.svc:5555) to collect the traces from the backend service.

### 4. Opentelemetry Native nginx module

The Opentelemetry Native nginx module can be found at the following location - https://docs.nginx.com/nginx/admin-guide/dynamic-modules/opentelemetry/

You can configure this module to send traces to the Opentelemetry collector by setting the otel_exporter in the nginx config.

This module helps configuring the nginx.conf to use different variables to customize the traces that can be sent to Discovery service. 
See here for a full list of [module-directives](https://docs.nginx.com/nginx/admin-guide/dynamic-modules/opentelemetry/#module-directives) configurations which can be set in the nginx config and used to explitly give some trace parameter values.

The datasource name of the APIs collected through this integration will be named after the attribute [otel_service_name](https://docs.nginx.com/nginx/admin-guide/dynamic-modules/opentelemetry/#otel_service_name) or a custom trace attribute [otel_span_attr](https://docs.nginx.com/nginx/admin-guide/dynamic-modules/opentelemetry/#otel_span_attr) which can be used to set datasource name via `ibm-api-discovery-datasource.name`

Below is an example of nginx config when using this module

**Sample nginx-config in a Kubernetes ConfigMap**

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
  namespace: nginx
data:
  nginx-cm.yaml: |
    load_module modules/ngx_otel_module.so;
    events {}
    http {
      otel_service_name nginx-native;
      otel_exporter {
        endpoint management-api-discovery-otel-collector.{namespace}.svc:5555;
      }
      otel_trace on;	
      server_tokens off;
      client_max_body_size 32m;
      server {
        listen 80;
        location / {
          # The following statement will proxy traffic to the upstream named Backend
          proxy_pass http://{backend-service}:{port}/;
        }
      }
    }
```
