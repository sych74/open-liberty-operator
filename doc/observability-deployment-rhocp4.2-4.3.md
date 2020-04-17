

# Observability with Open Liberty


The following document covers various topics for configuring and integrating your Open Liberty runtime with monitoring tools in the OpenShift cluster. This document has been tested with Red Hat OpenShift Container Platform (RHOCP) 4.2 and 4.3.

## How to deploy Kibana dashboards to monitor Open Liberty logging events  

Kibana dashboards are provided for visualizing events from the Open Liberty runtime.

To leverage these dashboards the logging events must be emitted in JSON format to standard-out. For information regarding how to configure an Open Liberty image with JSON logging please see [here](https://github.com/OpenLiberty/ci.docker#logging). 

Retrieve available Kibana dashboards tuned for Open Liberty logging events [here](https://github.com/OpenLiberty/open-liberty-operator/tree/master/deploy/dashboards/logging).

For information regarding how to import Kibana dashboards see the official documentation [here](https://www.elastic.co/guide/en/kibana/5.6/loading-a-saved-dashboard.html).

For effective management of logs emitted from applications, deploy your own Elasticsearch, Fluentd and Kibana (EFK) stack. For more information see the following [guide](https://kabanero.io/guides/app-logging-ocp-4-2/). 

Command-line JSON parsers, like JSON Query tool (jq), can be used to create human-readable views of JSON-formatted logs. In the following example, the logs are piped through grep to ensure that the message field is there before jq parses the line:

```
oc logs -f pod_name -n namespace | \
  grep --line-buffered message | \
  jq .message -r
```

## How to monitor your Liberty runtimes  

A MicroProfile Metrics enabled Open Liberty runtime is capable of tracking and observing metrics from the JVM and Open Liberty server as well as tracking MicroProfile Metrics instrumented within a deployed application. The tracked metrics can then be scraped by Prometheus and visualized with Grafana.

### MicroProfile Metrics 1.x and 2.x

The following steps outline how to manually create and modify a `server.xml` to add the `mpMetrics-2.2` feature and `monitor-1.0` feature that will be built as part of your Open Liberty image.  If you intend to configure with MicroProfile Metrics 1.1 you can use the `mpMetrics-1.1` feature in place of `mpMetrics-2.2`.

1. Create an XML file named `server_mpMetrics.xml` with the following contents and place it in the same directory as your Dockerfile:


```XML
<?xml version="1.0" encoding="UTF-8"?>
<server>
   <featureManager>
       <feature>mpMetrics-2.2</feature>
       <feature>monitor-1.0</feature>
   </featureManager>
   <quickStartSecurity userName="admin" userPassword="adminPwd"/>
</server>
```


The above `server.xml` configuration secures access to the server with basic authentication using the `<quickStartSecurity>` element. The `<quickStartSecurity>` is used in the above example for simplicity. When configuring your server you may wish to use a [basic registry](https://www.ibm.com/support/knowledgecenter/en/SSEQTP_liberty/com.ibm.websphere.wlp.doc/ae/twlp_sec_basic_registry.html) or an [LDAP registry](https://www.ibm.com/support/knowledgecenter/en/SSEQTP_liberty/com.ibm.websphere.wlp.doc/ae/twlp_sec_ldap.html) for securing authenticated access to your server. When using Prometheus to scrape data from the `/metrics` endpoint only the _Service Monitor_ approach can be configured to negotiate authentication with the Open Liberty server. 


2.    In your DockerFile, add the following line to copy the `server_mpMetrics.xml` file into the `configDropins/overrides` directory:


```DockerFile
COPY --chown=1001:0 server_mpMetrics.xml /config/configDropins/overrides/
```

### Enabling Prometheus to scrape data 


You will need to deploy Prometheus using the Prometheus Operator which will then utilize Service Monitors to monitor and scrape logs from target services. Details regarding how to deploy and configure Prometheus are [here](https://kabanero.io/guides/app-monitoring-ocp4.2/#deploy-prometheus-prometheus-operator).

### Visualizing your data with Grafana


There are IBM provided Grafana dashboards that leverage metrics from the JVM as well as from the Open Liberty runtime.  Details regarding how to deploy and configure Grafana are covered [here](https://kabanero.io/guides/app-monitoring-ocp4.2/#deploy-grafana).


You can find the access point of Grafana by running the following:


```bash
# oc get routes -n grafana
NAME          HOST/PORT                                      PATH      SERVICES      PORT      TERMINATION   WILDCARD
grafana-ocp   grafana-ocp-grafana.apps.9.37.135.153.nip.io             grafana-ocp   <all>     reencrypt     None
```

The `grafana` value is the namespace that you deploy Grafana to.

Sample Open Liberty Grafana dashboards are available for servers using either mpMetrics-1.x or mpMetrics-2.x [here](https://github.com/OpenLiberty/open-liberty-operator/tree/master/deploy/dashboards/metrics). Look in the featureManager section of the server.xml for either the mpMetrics feature or the umbrella microProfile feature to determine which dashboard to use.

|Umbrella Feature|  mpMetrics Feature | Dashboard|
|---|---|---|
|microProfile-1.2 - microProfile 2.2 |mpMetrics-1.x|ibm-websphere-liberty-grafana-dashboard.json|
|microProfile-3.0 |mpMetrics-2.x|       ibm-websphere-liberty-grafana-dashboard-metrics-2.0.json|

## How to use health info with service orchestrator  


MicroProfile Health allows services to report their readiness and liveness statuses (i.e UP if it is ready or alive and DOWN if its not ready/alive) through two endpoints. The Health data will be available on the `/health/live` and `/health/ready` endpoints for the liveness checks and for the readiness checks, respectively.
Readiness check allows third party services to know if the service is ready to process requests or not. e.g., dependency checks, such as database connections, application initialization, etc. 
Liveness check allows third party services to determine if the service is running. This means that if this procedure fails the service can be discarded (terminated, shutdown). It reports an individual service's status at the endpoints and indicates the overall status as UP if all the services are UP. A service orchestrator can then use these health check statuses to make decisions.


### MicroProfile Health 2.x

 The following steps outline how to manually create and modify a server.xml to add the mpHealth-2.x feature that will be built as part of your Open Liberty image.


Configure mpHealth-2.x feature in server.xml:


1.    Create an XML file named `server_mpHealth.xml`, with the following contents and place it in the same directory as your DockerFile:


```XML
<?xml version="1.0" encoding="UTF-8"?>
<server>
   <featureManager>
       <feature>mpHealth-2.1</feature>
   </featureManager>
   <quickStartSecurity userName="admin" userPassword="adminPwd"/>
</server>
```


2.    In your DockerFile, add the following line to copy the `server_mpHealth.xml` file into the `configDropins/overrides` directory:


```DockerFile
COPY --chown=1001:0 server_mpHealth.xml /config/configDropins/overrides/
```


## Configure the Kubernetes Liveness and Readiness Probes to use the MicroProfile Health REST Endpoints


Kubernetes provides liveness and readiness probes that are used to check the health of your containers. These probes can check certain files in your containers, check a TCP socket, or make HTTP requests.
  
Configure the readiness and liveness probe's fields to point to the MicroProfile Health REST endpoints.

### For mpHealth-2.x


Modify the readiness and liveness probe's fields to point to the MicroProfile Health REST endpoints, in the OpenLibertyApplication Custom Resource (CR):


```YAML
spec:
  applicationImage:
  ...
  readinessProbe:
    failureThreshold: 12
    httpGet:
      path: /health/ready
      port: 9443
      scheme: HTTPS
    initialDelaySeconds: 30
    periodSeconds: 2
    timeoutSeconds: 10
  livenessProbe:
    failureThreshold: 12
    httpGet:
      path: /health/live
      port: 9443
      scheme: HTTPS
    initialDelaySeconds: 30
    periodSeconds: 2
    timeoutSeconds: 10
...
```

## Enable storage for serviceability

Using the operator, you can enable the serviceability definition in your OpenLibertyApplication Custom Resource to create a PersistentVolumeClaim so that the logs from your application go to a single storage. Your cluster must either be configured to automatically bind the PersistentVolumeClaim to a PersistentVolume or you must bind it manually. 

The `serviceability.size` definition in the following example will automatically create a PersistentVolumeClaim with the specified size and is shared between all pods of the OpenLibertyApplication instance. For more information on the serviceability definition provided by the operator, please see the following [user guide](https://github.com/OpenLiberty/open-liberty-operator/blob/master/doc/user-guide.md#storage-for-serviceability).

Add the `serviceability.size` definition in your OpenLibertyApplication Custom Resource; the PersistentVolumeClaim should be created with the name `<application_name>-serviceability`:

```YAML
spec:
  applicationImage:
  ...
  serviceability:
    size: 1Gi
```