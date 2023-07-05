# Deploy a Learning Analytics stack to Kubernetes

[![CC BY-SA 4.0][cc-by-sa-shield]][cc-by-sa]

In this tutorial, you will learn to deploy a complete learning analytics stack
to Kubernetes in minutes! 🎉


**Disclaimer**

> To proceed with this tutorial, you need to be familiar in using the command
> line from your operating system. Being familiar with Kubernetes basic
> concepts would be a plus to completely understand every step of this
> tutorial.

## Prerequisites

-   A running Kubernetes cluster
-   `kubectl`, the official Kubernetes CLI: https://kubernetes.io/docs/reference/kubectl/
-   `helm`, the package manager for kubernetes: https://helm.sh/fr/docs/intro/install/
-   `curl`, the command line tool for transfering data with URLs: https://curl.se/

## Kubernetes 101

In this very first step of the tutorial, we will clone the Git project of the
workshop in a working directory:

```sh
# Go to your usual working directory
cd ${HOME}/work

# Clone this project, either with SSH:
git clone git@github.com:openfun/k8s-la-stack-tutorial.git
# or with HTTPS:
git clone https://github.com/openfun/k8s-la-stack-tutorial.git

# Go to our tutorial working directory
cd k8s-la-stack
```

We will now check that we are able to connect to the target Kubernetes cluster.
To do so, we need to download the Kubernetes cluster configuration file (_aka_
the Kubeconfig) and save it to our working directory:

```sh
# Move the configuration file to the current directory
#
# Nota bene: You need to adapt this example path to where your system stores
# downloaded files
mv ~/Download/kubeconfig.yml .
```

Now we need to define an environment variable pointing to the kubeconfig file
path before using the `kubectl` command:

```sh
# Define an environment variable pointing to the target cluster configuration file
export KUBECONFIG=${PWD}/kubeconfig.yml
```

> ⚠️ This `kubeconfig.yml` file should not be versionned or shared as it
> contains your credentials to connect to your Kubernetes cluster.

At this step, we should be able to send commands to the Kubernetes cluster
using the `kubectl` tool:

```sh
# Check cluster status
kubectl cluster-info

# List nodes of our cluster
kubectl get nodes
```

The response of this last command should look like the following:

```
NAME                                         STATUS   ROLES    AGE   VERSION
nodepool-08a7421c-46bf-4fe4-b6-node-1eb1ad   Ready    <none>   12m   v1.26.4
nodepool-08a7421c-46bf-4fe4-b6-node-8f3b8a   Ready    <none>   10m   v1.26.4
nodepool-08a7421c-46bf-4fe4-b6-node-d8d327   Ready    <none>   12m   v1.26.4
```

This tells us that our cluster has three active nodes running Kubernetes
`1.26.4` since few minutes.

We will now create our own Kubernetes namespace to work on:

```sh
# Generate a unique identifier for our namespace
export K8S_NAMESPACE="${USER:-user}-${RANDOM}"

# Check your namespace value
echo ${K8S_NAMESPACE}

# Create the namespace
kubectl create namespace ${K8S_NAMESPACE}

# Activate the namespace
kubectl config set-context --current --namespace=${K8S_NAMESPACE}
```

At this stage, we don't expect any pod to be running:

```sh
kubectl get pods
# Expected response is: No resources found in xxx-yyy namespace.
```

## Deploy applications

In this tutorial, we will deploy a full learning analytics stack composed of
the following components:

- **Learning Record Store (LRS)**, here [Ralph](https://github.com/openfun/ralph);
- **Database/datalake**, here [Elasticsearch](https://github.com/elastic/elasticsearch);
- **Dashboard system**, here [Superset](https://github.com/apache/superset).

Additionally, we will also deploy a Learning Management System (LMS), here
[Moodle](https://github.com/moodle/moodle).

The Moodle LMS generates learning traces and send them to the Ralph LRS in xAPI
format. The LRS stores learning traces in an Elasticsearch cluster that will be
set as the primary datasource of a generalist dashboarding system: Apache
Superset.

```mermaid
flowchart LR
    lms[Moodle - LMS] -- xAPI --> lrs[Ralph - LRS] --> data[(Elasticsearch - DB)] --> dashboard[Superset - dashboards]
```

### LMS: Moodle:tm:

Moodle:tm: can be installed using Helm in a single line of code:

```
helm install lms oci://registry-1.docker.io/bitnamicharts/moodle
```

> 💡 Note that it can take a few minutes before the service is up and running.

> 💡 Also note that, sometimes, on clusters with fewer resources, the Moodle
> pod happen to be stuck while the MariaDB pod is ready: either wait for the
> Moodle pod to restart, or restart it with the command:
> `kubectl delete pods -l app.kubernetes.io/name=moodle`



You can check deployment status using the `kubectl get pods -w` command. In a
similar way, the load balancer may take some time to be available, you can
check its status using the `kubectl get svc -w lms-moodle` command.

Once the service has an assigned external IP, you can get and store this IP
address (we will use it later):

```sh
# Define the MOODLE_IP variable
export MOODLE_IP=$(kubectl get svc lms-moodle --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")

# Display its value
echo "Moodle URL: http://${MOODLE_IP}/"
```

You can click on the displayed link from your terminal to discover your brand
new Moodle instance 🎉.

To login to your moodle instance, we need to fetch randomly generated password
using:

```sh
# Get user password
export MOODLE_PASSWORD=$(kubectl get secret lms-moodle -o jsonpath="{.data.moodle-password}" | base64 -d)

# Display credentials
echo "Moodle user: user"
echo "Moodle password: ${MOODLE_PASSWORD}"
```

Before configuring our Moodle instance, we need to deploy all other
applications from our learning analytics stack. Keep it up! 💪

### Data lake: Elasticsearch

In its recent releases, Elastic recommends to deploy its services using Custom
Resource Definitions (CRDs) installed via its official Helm chart. For the sake
of simplicity, we've installed those definitions cluster-wide so that all
namespaces can benefit from it.

You don't need to execute the commands below, but we share them for
information purpose:

```sh
#
# ⚠️  Don't execute the following commands ⚠️
#

# Add elastic official helm charts repository
helm repo add elastic https://helm.elastic.co

# Update available charts list
helm repo update

# Install the eck operator
helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
```

Since CRDs are already deployed cluster-wide, we will now be able to deploy a
two-nodes elasticsearch "cluster":

```sh
kubectl apply -f manifests/data-lake.yml
```

If you take a look at the data lake manifest, you will notice that official
Elastic CRDs eases the definition of a cluster:

```yaml
# manifests/data-lake.yml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: data-lake
spec:
  version: 8.8.1
  nodeSets:
    - name: default
      count: 2
      config:
        node.store.allow_mmap: false
      podTemplate:
        spec:
          containers:
          - name: elasticsearch
            env:
              - name: ES_JAVA_OPTS
                value: -Xms512m -Xmx512m
            resources:
              requests:
                memory: 512Mi
                cpu: 0.5
              limits:
                memory: 2Gi
                cpu: 2
```

Once applied, your elasticsearch pod should be running. You can check this using the following command:

```sh
kubectl get pods -w
```

We expect to see two pods called `data-lake-es-default-0` and
`data-lake-es-default-1`.

When our Elasticsearch cluster is up (this can take few minutes), you may
create the Elasticsearch index that will be used to store learning traces (xAPI
statements):

```sh
# Store elastic user password
export ELASTIC_PASSWORD="$(kubectl get secret data-lake-es-elastic-user -o jsonpath="{.data.elastic}" | base64 -d)"

# Execute an index creation request in the elasticsearch container
kubectl exec data-lake-es-default-0 --container elasticsearch -- \
    curl -ks -X PUT "https://elastic:${ELASTIC_PASSWORD}@localhost:9200/statements?pretty"
```

Our Elasticsearch cluster is all set. In the next section, We will now deploy
the [LRS](https://github.com/openfun/ralph).

### LRS: Ralph

Ralph is also distributed as a Helm chart that can be deployed with a single
line of code:

```sh
helm install --values charts/ralph/values.yaml lrs oci://registry-1.docker.io/openfuncharts/ralph
```

> 💡 You are now familiar with the procedure: you can check Ralph deployment
> using the `kubectl get pods -w` and `kubectl get svc -w lrs-ralph` commands.

Once deployed, you can get and store the IP address of the LRS:

```sh
# Define the LRS_IP variable
export LRS_IP=$(kubectl get svc lrs-ralph -o jsonpath="{.status.loadBalancer.ingress[0].ip}")

# Display its value
echo "LRS URL: http://${LRS_IP}/"
```

We will now check that we are able to query the LRS using the `curl` tool.

```sh
# Check configured user credentials
curl --user admin:password "http://${LRS_IP}/whoami"
```

> 💡 Alternatively you may choose to click on the LRS URL link from your
> terminal to open it with your default web browser. HTTP Basic Auth
> credentials are `admin` for the login and `password` for the password.

Edit `charts/ralph/values.yaml` to update the `elastic` user password: you
should substitute the `XXXXXXXXX` pattern by the value of the
`${ELASTIC_PASSWORD}` shell variable:

```yaml
# charts/ralph/values.yaml
envSecrets:
  # Replace the XXXXXXXXX value by elastic user password
  RALPH_BACKENDS__DATABASE__ES__HOSTS: https://elastic:XXXXXXXXX@data-lake-es-http:9200
# [...]
```

> 💡 To get the `elastic` user password, run the following command from your
> terminal: `echo "elastic user password: ${ELASTIC_PASSWORD}"`

To test our deployment, we will send 22 batches of 1k statements to the LRS:

```sh
# Send 22k xAPI statements in parallel 😎
\ls data/statements* | xargs -t -n 1 -P 10 -I {} bash -c " \
  gunzip -c {} | \
  curl -L \
    --user \"admin:password\" \
    -X POST \
    -H \"Content-Type: application/json\" \
    http://${LRS_IP}/xAPI/statements/ -d @- \
"
```

If everything went well, the LRS should respond with 22k UUIDs filling your
terminal with apparently random characters. 😅

Let's check what our statements look like by querying them to the LRS:

```sh
# Fetch stored statements
curl -L --user "admin:password" http://"${LRS_IP}"/xAPI/statements/
```

> 💡 Similarly to our previous remark, you may choose to click on the LRS URL
> link from your terminal to open it with your default web browser. HTTP Basic
> Auth credentials are `admin` for the login and `password` for the password.

### Data visualization: Apache Superset

Last but not least, we are now ready to deploy Superset, the shiny
datavisualization tool from the Apache Foundation.

This time, we need to add the official superset Helm repository locally and
then install its latest release:

```sh
# Add Superset official helm charts repository
helm repo add superset https://apache.github.io/superset

# Update available charts list
helm repo update

# Install the official superset chart
helm install --values charts/superset/values.yaml dataviz superset/superset
```

After one minute (or two), the `dataviz-superset` service should be up and
running and the load balancer should have an assigned IP.

**Questions**

1. What command should you run to check the service deployment status?
2. How will we get the load balancer IP?

⏰

![Mr Bean Waiting](https://media.giphy.com/media/QBd2kLB5qDmysEXre9/giphy-downsized.gif)

⏰

**Responses**

1. `kubectl get svc -w lrs-ralph`
2. `kubectl get svc dataviz-superset -o jsonpath="{.status.loadBalancer.ingress[0].ip}"`

You got it, congratz! 🎉

```sh
# Define the SUPERSET_IP variable
export SUPERSET_IP=$(kubectl get svc dataviz-superset -o jsonpath="{.status.loadBalancer.ingress[0].ip}")

# Display its value
echo "SUPERSET URL: http://${SUPERSET_IP}/"
```

All desired services for our complete learning analytics stack are now
deployed, w00t! It's time to configure our tools.

## Configure applications

### Moodle

#### Plugin: Logstore xAPI

To send xAPI statements from the LMS to the LRS, we need to install the
Logstore xAPI plugin from the Moodle plugins directory where it can be
downloaded: https://moodle.org/plugins/logstore_xapi

Once downloaded, go to "Site administration > Plugins > Install plugins". Once
the Zip package has been uploaded submit the form and follow the installation
workflow.

Configuration for the Logstore xAPI should look like:

```
LRS endpoint: http://xx.xx.xx.xx/xAPI/statements
Username: admin
LRS basic auth secret: password
Send statements by scheduled task? No
```

> Substitute the `xx.xx.xx.xx` pattern with the ${LRS_IP} value.

Once installed and configured, don't forget to enable the plugin by clicking in
the "Logging" section of the "Site administration > Plugins" page.

#### Generate a test course

We are now ready to generate events from the LMS and send them to the LRS,
but... we need a course for this!

From the "Site administration > Development" page, click on the "Debugging"
link from the development section. Select "DEVELOPER" for the "Debug messages"
and save changes.

Now that the developer mode is active, go back to the "Site administration >
Development" page and click on the "Make test course" link. Select a `S` size
course (it needs 30s to be generated) and give it a short/full name. After
seconds, you can navigate in the newly generated course. Feel free to test
various activities so that you generate learning traces we can explore in
Superset.

### Superset

Now it's time to have fun with learning traces! Login to your Superset instance
using `admin` as login and also as password. Go to "Settings > Data - Database
connections", add a new database of type "ElasticSearch (OpenDistro SQL)" with
the following SQLAlchemy connection URI:

```
elasticsearch+https://elastic:XXXXXXXXXX@data-lake-es-http:9200/?verify_certs=False
```

> Substiture the `XXXXXXXXXX` pattern with the `elastic` user password that is
> stored in the `${ELASTIC_PASSWORD}` variable.

You can now add a new dataset using the `statements` index from the
Elasticsearch database. Use this dataset to create new charts and dashboards.

---

This is it.

We are all set.

**You did it!** 🎉

![The Office Wow](https://media.giphy.com/media/tkApIfibjeWt1ufWwj/giphy.gif)

## License

This work is licensed under a
[Creative Commons Attribution-ShareAlike 4.0 International License][cc-by-sa].

[![CC BY-SA 4.0][cc-by-sa-image]][cc-by-sa]

[cc-by-sa]: http://creativecommons.org/licenses/by-sa/4.0/
[cc-by-sa-image]: https://licensebuttons.net/l/by-sa/4.0/88x31.png
[cc-by-sa-shield]: https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg
