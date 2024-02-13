# Istio installation For Opentelemetry Collector in Api Discovery
## Pre-req: Ensure istio is installed on your cluster 

Follow the istio getting started guide if you do not have a cluster with istio already installed.   

Deploy both the istio mesh and the sample application to test with.  
https://istio.io/latest/docs/setup/getting-started/  

The collector has been tested with `istioctl` version `1.19.x` & `1.20.x` 

## Configuring Istio to support the collector  

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
