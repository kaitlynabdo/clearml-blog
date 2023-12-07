# How-to: ClearML Enterprise on OpenShift

As the demand for artificial intelligence (AI) and its intelligent applications grows, technologies. Red Hat aims to meet some of those demands with OpenShift AI, as well as expanding our vast partner ecosystem to allow for greater AI capabilities. [In a previous blog](https://cloud.redhat.com/blog/a-guide-to-clearml-on-openshift), we walked through how to install the open source offering of ClearML. In this blog, we will dive into how to do a basic install of ClearML Enterprise on OpenShift.

## What is ClearML?

[ClearML](https://clear.ml/) offers a comprehensive suite of services that help customers revolutionize their landscapes of machine learning operations (MLOps). With a focus on end-to-end MLOps management, ClearML facilitates seamless experiment tracking, collaborative workflows, and efficient model deployment. This open source platform integrates popular machine learning frameworks, providing users with a versatile toolset for data management, scheduling and optimized computing, and model monitoring. ClearML is a pivotal solution for teams and enterprises that aim to expand their MLOps capabilities and optimize their machine learning processes in one consolidated platform. ClearML Enterprise offers customers additional features that allow customers to improve their MLOps with additional features like scalability, security, collaboration platforms, CI/CD tools, and more.

## Why OpenShift with ClearML Enterprise?

[ClearML Enterprise](https://clear.ml/clearml-enterprise) and OpenShift share many key features, providing greater abilities to grow, customize and monitor intelligent applications on consolidated platforms.

1\. **Scalability**

   - OpenShift and ClearML Enterprise have enhanced scalability features, allowing organizations to handle more complex machine learning workloads and intelligent applications.

2\. **Customization and Integration**

   - OpenShift and ClearML Enterprise offer more extensive customization options and better integration with existing enterprise infrastructure and ISV tools from the partner ecosystem.

3\. **Security and Compliance**

   - OpenShift and ClearML Enterprise prioritize and are dedicated to ensuring our products meet the stringent security and compliance requirements for corporate environments.

4\. **Advanced Analytics and Reporting**

   - While OpenShift has monitoring and analytics capabilities for your cluster, ClearML features enhanced analytics and reporting capabilities providing organizations with deeper insights into their machine learning processes.

## Prerequisites

-   OpenShift version 4.14
-   OC and Helm tools installed
-   ClearML Enterprise credentials

Before we begin, we'll make a new directory called clearml and move into it to keep this walkthrough organized. We will also set a *WILDCARD* variable with a wildcard URL. *Example: apps.ocp4.example.com*
```
$ mkdir clearml
$ cd clearml
$ export WILDCARD=<YOUR_OPENSHIFT_WILDCARD_APP_URL>
```

This above *WILDCARD* variable will be used later when we setup our Helm overrides configuration.

## Deploying the HELM Chart

We can deploy ClearML Enterprise via helm on our OpenShift cluster. With the credentials provided to you from the ClearML team, pull the helm chart to add the repository and update it.


```
$ helm repo add allegroai-enterprise <CLEAR_ML_HELM_CHART_URL> --username <YOUR_USERNAME> --password <YOUR_PASSWORD>

"allegroai-enterprise" has been added to your repositories
```

```
$ helm repo update

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "allegroai-enterprise" chart repository
Update Complete. ⎈Happy Helming!⎈
```

Now that we have our repository, let's create a new project for ClearML in our OpenShift Cluster.

```
$ oc new-project clearml

Now using project "clearml" on server.
You can add applications to this project with the 'new-app' command. For example, try:
    oc new-app rails-postgresql-example
to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:
    kubectl create deployment hello-node --image=registry.k8s.io/ -- /agnhost serve-hostname
```

Create a file called *override.yml* with the following information to override the default configurations in our OpenShift cluster with our own. You will need image credentials to access the ClearML private docker registry and insert that password under *imageCredentials*.
```
$ cat <<EOF >> /tmp/override.yaml
imageCredentials:
  password: "<CLEARML_PRIV_REGISTRY_CREDENTIALS>"

elasticsearch:
  rbac:
    create: true
    serviceAccountName: "clearml-elastic"

apiserver:
  podSecurityContext:
    runAsUser: 0
  service:
    type: ClusterIP
  additionalConfigs:
    apiserver.conf: |
      auth {
        fixed_users {
          enabled: true
          pass_hashed: false
          users: [
            {
            username: "testuser"
            password: "testpassword"
            name: "Test User"
            admin: true
           },
          ]
         }
        }
  ingress:
    # -- Enable/Disable ingress
    enabled: true
    # -- ClassName (must be defined if no default ingressClassName is available)
    ingressClassName: ""
    # -- Ingress hostname domain
    hostName: "clearml-enterprise-apiserver-clearml.$WILDCARD"
    # -- Reference to secret containing TLS certificate. If set, it enables HTTPS on ingress rule.
    tlsSecretName: ""
    # -- Ingress annotations
    annotations: {}
    # -- Ingress root path url
    path: "/"

fileserver:
  service:
    type: ClusterIP
  ingress:
    # -- Enable/Disable ingress
    enabled: true
    # -- ClassName (must be defined if no default ingressClassName is available)
    ingressClassName: ""
    # -- Ingress hostname domain
    hostName: "clearml-enterprise-fileserver-clearml.$WILDCARD"
    # -- Reference to secret containing TLS certificate. If set, it enables HTTPS on ingress rule.
    tlsSecretName: ""
    # -- Ingress annotations
    annotations: {}
    # -- Ingress root path url
    path: "/"
  # -- File Server extra envrinoment variables

webserver:
  service:
    type: ClusterIP
  ingress:
    # -- Enable/Disable ingress
    enabled: true
    # -- ClassName (must be defined if no default ingressClassName is available)
    ingressClassName: ""
    # -- Ingress hostname domain
    hostName: "clearml-enterprise-webserver-clearml.$WILDCARD"
    # -- Reference to secret containing TLS certificate. If set, it enables HTTPS on ingress rule.
    tlsSecretName: ""
    # -- Ingress annotations
    annotations: {}
    # -- Ingress root path url
    path: "/"
EOF
```

We need to change the security context constraints for some of our users. Currently, we need to allow broader permissions to deploy ClearML Enterprise, so we'll allow anyuid and privileged permissions to the following users. ClearML does have a non root option for security, but due to a bug in helm at the time of this blog, we had to allow broader permissions.

```
$ oc adm policy add-scc-to-user anyuid -z clearml-apiserver
$ oc adm policy add-scc-to-user anyuid -z clearml-enterprise-mongodb
$ oc adm policy add-scc-to-user anyuid -z clearml-enterprise-redis
$ oc adm policy add-scc-to-user anyuid -z default
$ oc adm policy add-scc-to-user privileged -z clearml-elastic
```
```
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "clearml-apiserver"
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "clearml-enterprise-mongodb"
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "clearml-enterprise-redis"
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "default"
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "clearml-elastic"
```

Finally, we are now able to deploy the helm chart to our cluster with the custom configurations we made with *override.yaml*. It should take about 5 minutes for the helm chart to install and deploy, but many vary depending on your environment's resources.

```
$ helm install clearml-enterprise allegroai-enterprise/clearml-enterprise -f /tmp/override.yaml

NAME: clearml-enterprise
LAST DEPLOYED: Wed Nov 22 10:13:26 2023
NAMESPACE: clearml
STATUS: deployed
REVISION: 1
NOTES:
1\. Get the application URL:
  export POD_NAME=$(kubectl get pods --namespace clearml -l "app.kubernetes.io/name=clearml-enterprise,app.kubernetes.io/instance=clearml-enterprise" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace clearml $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace clearml port-forward $POD_NAME 8080:$CONTAINER_PORT
Accessing Route to ClearML Enterprise Dashboard
```

Now that we have finally deployed the helm chart, we can ClearML Enterprise. To do so, we need to access the route to our ClearML web server. To do so, we need to first expose the API server, web server, and file server.

```
$ oc expose svc clearml-enterprise-apiserver
$ oc expose svc clearml-enterprise-webserver
$ oc expose svc clearml-enterprise-fileserver
```
```
route.route.openshift.io/clearml-enterprise-apiserver exposed
route.route.openshift.io/clearml-enterprise-webserver exposed
route.route.openshift.io/clearml-enterprise-fileserver exposed
```

To access the web server address, we need to view the routes we just exposed. Copy and paste the web server address into your internet browser.

```
$ oc get routes

clearml-enterprise-apiserver    clearml-enterprise-apiserver-clearml.apps.ocp4.example.com         clearml-enterprise-apiserver    8008                 None
clearml-enterprise-fileserver   clearml-enterprise-fileserver-clearml.apps.ocp4.example.com         clearml-enterprise-fileserver   8081                 None
clearml-enterprise-webserver    clearml-enterprise-webserver-clearml.apps.ocp4.example.com           clearml-enterprise-webserver    8080                 None
```

## ClearML Dashboard

Once you have access to the web server, log in with:

Username: *testuser*

Password: *testpassword*

This username and password combination was set in the override.yaml under apiServer, so user access can be customized.


<img width="1231" alt="Screenshot 2023-11-22 at 10 20 22 AM" src="https://github.com/kaitlynabdo/clearml-blog/assets/45447032/9e9ba663-da3f-43eb-a87f-123ab7c035da">


Once you have logged in, you will reach the ClearML dashboard.

<img width="1223" alt="Screenshot 2023-11-22 at 10 22 46 AM" src="https://github.com/kaitlynabdo/clearml-blog/assets/45447032/5f3abd3d-6e28-4953-8852-eaa776376690">


That's a wrap! Now you know how to deploy ClearML on your OpenShift cluster. Now you can try out the features and capabilities with some [examples](https://clear.ml/docs/latest/docs/guides/) with the sample content in ClearML to

If you would like to try out OpenShift, you can with our [Developer Sandbox](https://developers.redhat.com/developer-sandbox) using a free trial. For more information on OpenShift and our AI initiatives, and ClearML, please visit the following pages:

-   [OpenShift](https://www.redhat.com/en/technologies/cloud-computing/openshift)

-   [OpenShift AI](https://www.redhat.com/en/technologies/cloud-computing/openshift/openshift-ai)

-   [ClearML Guide to OpenShift](https://cloud.redhat.com/blog/a-guide-to-clearml-on-openshift)

-   [ClearML Documentation](https://clear.ml/docs/latest/)
