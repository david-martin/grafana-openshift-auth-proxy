# Grafana OAuth integration with OpenShift using an Auth Proxy

## Example running on OpenShift

NOTE: Uses https://github.com/openshift/oauth-proxy

**Create a new project in OpenShift**

```bash
export GRAFANA_PROJECT=grafana-oauth-proxy
oc new-project $GRAFANA_PROJECT
```

**Create Grafana & oauth proxy resources**

```bash
oc process -f grafana_oauth_proxy_template.yaml | oc create -f -
```

**Go to the proxy route & login with OpenShift credentials**

```
oc get route proxy --template "https://{{.spec.host}} "
```

You should initially see a 'Sign in with an OpenShift account' button.
Click it, then sign into OpenShift.
You should be redirected to Grafana and be logged in.

![grafana dashboard](grafana_dashboard.png?raw=true "Grafana Dashboard")

## Alternative Approach: Local Docker auth proxy

**Assumptions**

* Docker installed
* Locally running OpenShift cluster available at https://192.168.37.1:8443

**Run Grafana**

```bash
docker run -i -v $(pwd)/grafana.ini:/etc/grafana/grafana.ini --name grafana grafana/grafana
export GRAFANA_DOCKER_IP=$(docker inspect --format '{{.NetworkSettings.IPAddress}}' grafana)
```

**Create a new project in OpenShift**

```bash
export GRAFANA_PROJECT=my-grafana-project
oc new-project $GRAFANA_PROJECT
```

**Create the ServiceAccount OAuth Client**

```bash
oc create -f ./serviceaccount.yaml
```

**Get the OAuth Client secret**
```bash
oc sa get-token grafana-client > ~/work/auth-proxy-stuff/secret/oauth-secret
```

**Run the OpenShift Auth Proxy**

```bash
openshift-auth-proxy \
  --listen-port 8080 \
  --backend http://$GRAFANA_DOCKER_IP:3000 \
  --auth-mode oauth2 \
  --master-url https://192.168.37.1:8443 \
  --public-master-url https://192.168.37.1:8443 \
  --user-header X-WEBAUTH-USER \
  --master-ca ./ca-bundle.crt \
  --server-cert ./serving.crt \
  --server-key ./serving.key \
  --transform user_header \
  --oauth-id system:serviceaccount:$GRAFANA_PROJECT:grafana-client \
  --debug
```

**Sign in**

Go to https://localhost:8080 & sign in with your OpenShift credentials

