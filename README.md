# Liberty mpTelemetry feature

This document is a memo from when I experimented with the Liberty mpTelemetry-2.0 feature. With this feature, liberty automatically detects REST service running on it and sends metrics to OpenTelemetry collector. Then, I configured the collector to export data to instana.  

1. Provision OCP 4.17.0.

2. Install Red Hat build of OpenTelemetry operator

3. Create a project for OpenTelemetry Collector

4. With an IBMID, you can use Instana trial for 14 days. After registration, you will receive the following main including "Agent Key". This is needed to integrate OpenTelemetry collector with your Instana service.   
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/bcaf8f49-258e-470f-a610-4e5c1a228eea">

5. In the Instana console, check your reagion.  
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/e0829006-45d1-48c1-963b-5ac190037754">
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/fa04ffab-9d7b-4b6b-9076-d8d18431871d">

6. check the endpoint of the reagion.
https://www.ibm.com/docs/en/instana-observability/current?topic=instana-backend
<img width="400" alt="image" src="https://media.github.ibm.com/user/24674/files/d94957c3-c7ff-4a0a-a4b8-0e0758ed4250">

7. create a OpenTelemetry Collector with the following YAML in the project created at the step 3.   

```
kind: OpenTelemetryCollector
apiVersion: opentelemetry.io/v1beta1
metadata:
  name: otel
  namespace: e30532
spec:
  mode: "deployment"
  ingress:
    type: route
    route:
      termination: "passthrough"
  config:
    exporters:
      otlp:
        endpoint: otlp-green-saas.instana.io:4317
        headers: 
          x-instana-key: YOUR_AGENT_KEY
    receivers:
      otlp:
        protocols:
          grpc:
            tls:
          http:
            tls:
    service:
      pipelines:
        traces:
          exporters: [otlp]
          receivers: [otlp]
        metrics:
          exporters: [otlp]
          receivers: [otlp]
        logs:
          exporters: [otlp]
          receivers: [otlp]

```
<img width="999" alt="image" src="https://media.github.ibm.com/user/24674/files/4ac2ff3c-8128-4c2c-804d-97c06ddd4cda">.  

Note: Initially, I tried to let a liberty running on on-prem send the data to the collector via route. However, it failed with the following error.   
```
[10/23/24, 0:59:10:044 PDT] 00000043 io.opentelemetry.exporter.internal.grpc.GrpcExporter         W Failed to export logs. Server responded with gRPC status code 2. Error message: 
[10/23/24, 0:59:15:038 PDT] 00000055 io.opentelemetry.exporter.internal.grpc.GrpcExporter         W Failed to export metrics. Server responded with gRPC status code 2. Error message: 
```
Therefore, I let the liberty running in the same namespace access the opentelemetry collector service (otel-collector).


8. create a REST application. There are three endpoints. 

myservice : a simple REST
myservice2 : This measures the processing time of a mock database query using span
myservice3 : This always fails due to ArithmeticException.   

```
package my.rest;

import jakarta.ws.rs.ApplicationPath;
import jakarta.ws.rs.core.Application;

@ApplicationPath("/api")
public class MyApplication extends Application{

}
```

```
package my.rest;

import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;

@Path("/myservice")
public class MyService {
	@GET
    @Produces(MediaType.TEXT_PLAIN)
    public String sayHello() {
        try {
			Thread.sleep(5*1000);
		} catch (Exception e) {
			e.printStackTrace();
		}
        return "Hello!";
    }
}
```

```
package my.rest;

import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Scope;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;


@Path("/myservice2")
public class MyService2 {
	@Inject
	private Tracer tracer;

	@Inject
	private Span newSpan;

	
	@GET
    @Produces(MediaType.TEXT_PLAIN)
    public String doDBQuery() {
		newSpan = tracer.spanBuilder("QueryDatabase").startSpan();
		try (Scope s = newSpan.makeCurrent()) {
	        try {
				Thread.sleep(10*1000);
			} catch (Exception e) {
				e.printStackTrace();
			}
	    } finally {
	        newSpan.end();
	    }
		return "Hello!";
    }
}
```

```
package my.rest;

import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Scope;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;


@Path("/myservice3")
public class MyService3 {

	@GET
    @Produces(MediaType.TEXT_PLAIN)
    public String sayHello() {
		int i = 1/0;
		return "Hello!";
    }
}
```


9. build a image and deploy it on OCP.

```
[root@c89289v1 skill]# pwd
/root/skill
[root@c89289v1 skill]# ls -l
total 24
-rw-r--r-- 1 root root  407 Oct 22 23:47 Dockerfile
-rw-r--r-- 1 root root 4032 Oct 23 00:25 MyRest.ear
-rw-r--r-- 1 root root  132 Oct 22 23:47 bootstrap.properties
-rw-r--r-- 1 root root  758 Oct 22 23:55 server.xml
[root@c89289v1 skill]# cat Dockerfile 
FROM icr.io/appcafe/open-liberty
USER root
COPY server.xml /config/server.xml
COPY bootstrap.properties /config/bootstrap.properties
COPY MyRest.ear /config/apps/MyRest.ear
RUN chown -R 1001.0 /config /opt/ol/wlp/usr/servers/defaultServer /opt/ol/wlp/usr/shared/resources && chmod -R g+rw /config /opt/ol/wlp/usr/servers/defaultServer /opt/ol/wlp/usr/shared/resources
USER 1001
RUN configure.sh
EXPOSE 9080
[root@c89289v1 skill]# cat bootstrap.properties 
otel.sdk.disabled=false
otel.traces.exporter=otlp
otel.exporter.otlp.endpoint=http://otel-collector:4317/
otel.service.name=liberty
[root@c89289v1 skill]# cat server.xml 
<?xml version="1.0" encoding="UTF-8"?>
<server description="Default server">

    <!-- Enable features -->
    <featureManager>
       <feature>restfulWS-3.1</feature>
       <feature>mpTelemetry-2.0</feature>
    </featureManager>

    <!-- To allow access to this server from a remote client host="*" has been added to the following element -->
    <httpEndpoint id="defaultHttpEndpoint"
                  host="*"
                  httpPort="9080"
                  httpsPort="9443" />
    <applicationManager autoExpand="true"/>    
    <application id="MyRest" location="MyRest.ear" type="ear" context-root="/MyRestWeb">
      <classloader apiTypeVisibility="+third-party"/>
    </application>
    <mpTelemetry source="message, trace, ffdc"/>
</server>
[root@c89289v1 skill]# 

podman build -t myrest .

podman tag myrest:latest 'default-route-openshift-image-registry.apps.xxx.ibm.com/e30532/myrest'

oc login --token=sha256~9xxxxxxxxxxx0 --server=https://api.xxx.ibm.com:6443

vi /etc/containers/registries.conf
[[registry]]
location = "default-route-openshift-image-registry.apps.xxx.ibm.com"
insecure = true
 
podman login -u kubeadmin -p $(oc whoami -t) https://default-route-openshift-image-registry.apps.xxx.ibm.com
podman push default-route-openshift-image-registry.apps.xxx.ibm.com/e30532/myrest
oc new-app -i myrest
oc expose service/myrest
```

10. let's check the instana console after accessing the REST services.
You will see liberty detects the JAX-RS endpoints automatically and sends their static data.   
<img width="600" alt="image" src="https://media.github.ibm.com/user/24674/files/a222a750-1083-4f5b-8b89-57afefba042e">.  
<img width="600" alt="image" src="https://media.github.ibm.com/user/24674/files/a2d9e811-3bc5-4d6c-b165-c64705857a4b">.  
<img width="600" alt="image" src="https://media.github.ibm.com/user/24674/files/4d66ec26-94d9-49c5-b578-e626b0dc2427">.  
<img width="600" alt="image" src="https://media.github.ibm.com/user/24674/files/cd08fbee-e381-465c-9f0d-c7598b77a216">.  
<img width="600" alt="image" src="https://media.github.ibm.com/user/24674/files/0c774b99-79fb-4815-8971-6e92376f99ef">.  
With the analytics feature of Instana, you will see the span named QueryDatabase is the bottleneck.     
<img width="600" alt="image" src="https://media.github.ibm.com/user/24674/files/4e459aae-01e0-4710-a43c-f80d4aee1297">.  


Hopefully, this document gives you a brief overview of the mpTelemetry feature.   
