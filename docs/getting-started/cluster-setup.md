# Setting Up a Cluster
You can setup a cluster in [Kubernetes](http://kubernetes.io/docs/whatisk8s/) or Jenkins.

## Setting Up a Cluster on AWS using Kubernetes

### Prerequisites
- [kubectl](http://kubernetes.io/docs/user-guide/prereqs/)
- an [AWS](http://aws.amazon.com) account
- [AWS CLI](https://aws.amazon.com/cli/)

### Create your cluster
A cluster consists of a master API server and a set of worker VMs called nodes.

#### Using get-kube
To start up your cluster, run:

```bash
#Using wget
export KUBERNETES_PROVIDER=aws; wget -q -O - https://get.k8s.io | bash
#Using cURL
export KUBERNETES_PROVIDER=aws; curl -sS https://get.k8s.io | bash
```

This process takes about 5 to 10 minutes. User credentials and security tokens are written in `~/.kube/config`.
By default, the script will provision a new VPC and a 4 node k8s cluster in `us-west-2a` (Oregon) with EC2 instances running on Debian.

NOTE: You can override the variables defined in `config-default.sh` to change this behavior. See the [Getting Started with Kubernetes guide](http://kubernetes.io/docs/getting-started-guides/aws/) for more details.



### Create a Service
A Kubernetes [Service](http://kubernetes.io/docs/user-guide/services/) is a REST object, similar to a Pod. A Service is an abstraction which defines a logical set of Pods and a policy by which to access them - sometimes called a micro-service. The set of Pods targeted by a Service is (usually) determined by a Label Selector. We will create a Service because persist while Pods are dynamically created and destroyed regularly.

To create a Service, we need to create a `service.yaml` file:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sdapi
  labels:
    app: screwdriver
    tier: api
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: screwdriver
    tier: api
```

Run the kubectl create command:
```bash
$ kubectl create -f service.yaml
```


### Setup secrets
You can create a secret object in a file first, in json or yaml format, and then create that object.
First, you will need to acquire your secrets.

#### Kubernetes secrets
Kubectl can be used to see your [Kubernetes secrets](http://kubernetes.io/docs/user-guide/secrets/walkthrough/).
To get your k8s `host`, run:
```bash
$ kubectl cluster-info

Kubernetes master is running at https://55.555.55.55
```

To get the k8s `<DEFAULT_TOKEN_NAME>`, run:
```bash
$ kubectl get secrets

NAME                      TYPE                                  DATA      AGE
datastore-dynamodb        Opaque                                2         19d
db-user-pass              Opaque                                2         49d
default-token-abc55       kubernetes.io/service-account-token   3         50d
git-token                 Opaque                                1         42d
mysecret                  Opaque                                2         49d
npm-token                 Opaque                                1         42d
plugin-login-secrets      Opaque                                4         44d
screwdriver-job-secrets   Opaque                                1         43d
```

The `<DEFAULT_TOKEN_NAME>` will be listed under `Name` when the `Type` is `kubernetes.io/service-account-token`.

To get your k8s `token`, run:
```bash
$ kubectl describe secrets <DEFAULT_TOKEN_NAME>  # default-token-abc55 in this example
```

#### DynamoDB secrets
To get your `accessKeyId` and `secretAccessKey`, navigate to [IAM](https://console.aws.amazon.com/iam)
in your AWS console. Click on Users, and create a Screwdriver user with `<USERNAME>`. We recommend installing using an account which has full access to the AWS APIs. Select the `Security Credentials` tab and click `Create Access Key`. Download the file and keep note of those values.

#### Screwdriver secrets
You should create your own `default.yaml` file and fill out the constants with your own secrets.
```yaml
---
login:
    # A private key uses for signing jwt tokens. Can be anything
    jwtPrivateKey: THIS-IS-AN-INSECURE-KEY
    # The client id used for OAuth with github. Look up GitHub OAuth for details
    # https://developer.github.com/v3/oauth/
    oauthClientId: YOU-PROBABLY-WANT-SOMETHING-HERE
    # The client secret used for OAuth with github
    oauthClientSecret: AGAIN-SOMETHING-HERE-IS-USEFUL
    # A password used for encrypting session, and OAuth data.
    # **Needs to be minimum 32 characters**
    password: WOW-ANOTHER-INSECURE-PASSWORD!!!
    # A flag to set if the server is running over https.
    # Used as a flag for the OAuth flow
    https: false

httpd:
    port: 8080

datastore:
    plugin: dynamodb
    imdb:
        # File to read/write the database to
        filename: ./database.json
    dynamodb:
        # AWS Access Key ID
        accessKeyId: WHAT-A-LEGITIMATE-LOOKING-KEY-ID
        # AWS Secret Access Key
        secretAccessKey: TOTALLY-REAL-LOOKING-AWS-KEYS

executor:
    plugin: k8s
    k8s:
        # The host or IP of the kubernetes cluster
        host: kubernetes
        # The jwt token used for authenticating kubernetes requests
        token: NOT-A-REAL-JWT-TOKEN

webhooks:
    github:
        # Secret to add to GitHub webhooks so that we can validate them
        secret: SUPER-SECRET-SIGNING-THING
```

Create the secret using `kubectl create`:
```bash
$ kubectl create -f ./secret.yaml
```


### Deploy Screwdriver
A [Deployment](http://kubernetes.io/docs/user-guide/deploying-applications/) provides declarative updates for Pods. You can define Deployments to create new resources, or replace existing ones by new ones.

Create a `deployment.yaml` file:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sdapi
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: screwdriver
        tier: api
    spec:
      containers:
      - name: screwdriver-api
        image: screwdrivercd/screwdriver
        ports:
        - containerPort: 8080
        env:
          - name: DATASTORE_PLUGIN
            value: dynamodb
          - name: DATA_DIR
            value: /opt/screwdriver/data
          - name: SECRET_JWT_PRIVATE_KEY
            valueFrom:
              secretKeyRef:
                name: plugin-login-secrets
                key: jwtprivatekey
          - name: SECRET_OAUTH_CLIENT_ID
            valueFrom:
              secretKeyRef:
                name: plugin-login-secrets
                key: oauthclientid
          - name: SECRET_OAUTH_CLIENT_SECRET
            valueFrom:
              secretKeyRef:
                name: plugin-login-secrets
                key: oauthclientsecret
          - name: SECRET_PASSWORD
            valueFrom:
              secretKeyRef:
                name: plugin-login-secrets
                key: password
          - name: K8S_TOKEN
            valueFrom:
              secretKeyRef:
                name: <DEFAULT_TOKEN_NAME>
                key: token
          - name: DATASTORE_DYNAMODB_ID
            valueFrom:
              secretKeyRef:
                name: datastore-dynamodb
                key: id
          - name: DATASTORE_DYNAMODB_SECRET
            valueFrom:
              secretKeyRef:
                name: datastore-dynamodb
                key: secret
```

Run your deployment:
```bash
$ kubectl create -f deployment.yaml
```

### View your pods
A Kubernetes [pod](http://kubernetes.io/docs/user-guide/pods/) is a group of containers, tied together for the purposes of administration and networking. It can contain a single container or multiple.

To view the pod created by the deployment, run:

```bash
$ kubectl get pods
```

To view the stdout / stderr from a pod, run:

```bash
$ kubectl logs <POD-NAME>
```
