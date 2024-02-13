# Opentelemetry Collector For Api Discovery

This document describes the set up details for the Istio based OpenTelemetry Collector For Api Discovery.   

## Pre-req: Ensure istio is installed on your cluster 

Follow the istio getting started guide if you do not have a cluster with istio already installed.   

Deploy both the istio mesh and the sample application to test with.  
https://istio.io/latest/docs/setup/getting-started/  

The collector has been tested with `istioctl` version `1.19.x` & `1.20.x`  

## Pre-req: Ensure helm is installed  

See here for details  
https://helm.sh/docs/intro/install/  

## Configuring Istio and deploying the collector  

### Update the Istio mesh config  
Firstly a reference to the discovery collector (which will be deployed in the suqsequent steps) will need to be registered in the istio MeshConfig to send trace data to the discovery service.  
See the Provider Selection section for some details: https://istio.io/latest/docs/tasks/observability/telemetry/#provider-selection    
There is an example of the patch that will need to be configured on the mesh in the [apidiscovery/patches](apidiscovery/patches) directory.   
This is a complete example of what the `istio` configmap could look like but the salient lines from the example which need to be added to your own configuration under extensionProviders are the following   
```
    extensionProviders:
      - name: "discoveryOtelCollector"
        opentelemetry:
          service: "management-api-discovery-otel-collector.istio-system.svc.cluster.local"
          port: 5555
```
These can be added by doing an edit on the `istio` configmap which will be in the namespace where istio is deployed.  
```
e.g.
kubectl edit configmap istio -n <istio-system>
```
Or if you have deployed istio via the IstioOperator you can configure the same via the MeshConfig  
See here for more details: https://istio.io/v1.10/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig  

### Use the Helm chart to deploy the collector.  

### Use the Helm chart to deploy the collector.  

In the helm directory, update the [values.yaml](apidiscovery/values.yaml) file based on your apiconnect provider org, authentication and istio set up  

The following parameters require updates:  

 - `discovery.datasource_name` - Optional field to set the datasource name for all APIs discovered by the collector. If datasource name is empty, the namespace of the collected API will be used.  
 - `discovery.apic_host_domain` - Domain name of the ApiConnect instance where discovered APIs will be sent.<br /> &nbsp; Example : `us-east-a.apiconnect.automation.ibm.com`  
 - `discovery.provider_org` - The provider org name of the apiconnect manager  
 - `discovery.apikey` - An API Key can be obtained from the api-manager for the user who has permission to create an API.  
&nbsp; Get the API key from the APIC Manager using a link of the following structure - `http://{api-host}/manager/auth/manager/sign-in/?from=TOOLKIT` (typically used with an OIDC user registry like IBM Verify). 
The apikey will be added to a kubernetes secret as part of the deployment and then mounted on the collector deployment pod.  
- `istio_namespace`: The namespace where istio has been deployed on your cluster. As required by istio's telemetry integration the collector pod will be deployed here.      
- `telemetry_namespace`: This is where the istio Telemetry CR will be deployed. By default the values file sets this to `istio-system` as the root configuration namespace to provide mesh level collection configuration. This can be customized as you wish based on your collection requirements regarding the deployed applications you want to be discovered. See here for further details https://istio.io/latest/docs/reference/config/telemetry/.  
- `images.api_discovery_collector`: As new versions of the collectors are released updating this property will enable the upgrade of the collector deployment. Note: collectors will require updates to ensure they remain compatible with the discovery service. Details of these updates will be available in this repository.
- `logging.log_level`: Default log_level is `info`. For debug purposes the log level can be increased to debug if needed.  

Once you have made the required updates to the values.yaml file you can deploy the collector.  

Run the following command to deploy the collector  
```
helm template . | kubectl apply -f -
```
Output should be as follows which indicate that the required Kubernetes Reources have been deployed  

```
helm template . | kubectl apply -f -
secret/api-discovery-secret created
service/management-api-discovery-otel-collector created
deployment.apps/management-api-discovery-otel-collector created
telemetry.telemetry.istio.io/api-discovery-otel created
```

Once the collector has been deployed any istio instructmented pods which have traffic running through envoy will begin to be discovered and sent to the apiconnect discovery service.  

## Uninstalling the collector

The collector can be uninstalled using the following comman.  

```
helm template . | kubectl delete -f -
```
