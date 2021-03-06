# Heptio Authenticator for AWS

A tool for using AWS IAM credentials to authenticate to a Kubernetes cluster.

## Why do I want this?
If you are an administrator running a Kubernetes cluster on AWS, you already need to manage AWS credentials for provisioning and updating the cluster.
By using Heptio Authenticator for AWS, you avoid having to manage a separate credential for Kubernetes access.
AWS IAM also provides a number of nice properties such as an out of band audit trail (via CloudTrail) and 2FA/MFA enforcement.

If you are building a Kubernetes installer on AWS, Heptio Authenticator for AWS can simplify your bootstrap process.
You won't need to somehow smuggle your initial admin credential securely out of your newly installed cluster.
Instead, you can create a dedicated `KubernetesAdmin` role at cluster provisioning time and set up Authenticator to allow cluster administrator logins.

## How do I use it?
Assuming you have a cluster running in AWS and you want to add Heptio Authenticator for AWS support, you need to:
 1. Create an IAM role you'll use to identify users.
 2. Run the Authenticator server as a DaemonSet.
 3. Configure your API server to talk to Authenticator.
 4. Set up kubectl to use Authenticator tokens.

### 1. Create an IAM role
First, you must create one or more IAM roles that will be mapped to users/groups inside your Kubernetes cluster.
The easiest way to do this is to log into the AWS Console:
 - Choose the "Role for cross-account access" / "Provide access between AWS accounts you own" option.
 - Paste in your AWS account ID number (available in the top right in the console).
 - Your role does not need any additional policies attached.

This will create an IAM role with no permissions that can be assumed by authorized users/roles in your account.
Note the Amazon Resource Name (ARN) of your role, which you will need below.

You can also do this in a single step using the AWS CLI instead of the AWS Console:
```sh
# get your account ID
ACCOUNT_ID=$(aws sts get-caller-identity --output text --query 'Account')

# define a role trust policy that opens the role to users in your account (limited by IAM policy)
POLICY=$(echo -n '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':root"},"Action":"sts:AssumeRole","Condition":{}}]}')

# create a role named KubernetesAdmin (will print the new role's ARN)
aws iam create-role \
  --role-name KubernetesAdmin \
  --description "Kubernetes administrator role (for Heptio Authenticator for AWS)." \
  --assume-role-policy-document "$POLICY" \
  --output text \
  --query 'Role.Arn'
```

You can also skip this step and use:
 - An existing role (such as a cross-account access role).
 - An IAM user (see `mapUsers` below).
 - An EC2 instance or a federated role (see `mapRoles` below).

### 2. Run the server
The server is meant to run on each of your master nodes as a DaemonSet with host networking so it can expose a localhost port.

For a sample ConfigMap and DaemonSet configuration, see [`example.yaml`](./example.yaml).

#### (Optional) Pre-generate a certificate, key, and kubeconfig
If you're building an automated installer, you can also pre-generate the certificate, key, and webhook kubeconfig files easily using `heptio-authenticator-aws init`.
This command will generate files and place them in the configured output directories.

You can run this on each master node prior to starting the API server.
You could also generate them before provisioning master nodes and install them in the appropriate host paths.

If you do not pre-generate files, `heptio-authenticator-aws server` will generate them on demand.
This works but requires that you restart your Kubernetes API server after installation.

### 3. Configure your API server to talk to the server
The Kubernetes API integrates with Heptio Authenticator for AWS using a [token authentication webhook](https://kubernetes.io/docs/admin/authentication/#webhook-token-authentication).
When you run `heptio-authenticator-aws server`, it will generate a webhook configuration file and save it onto the host filesystem.
You'll need to add a single additional flag to your API server configuration:
```
--authentication-token-webhook-config-file=/etc/kubernetes/heptio-authenticator-aws/kubeconfig.yaml
```

On many clusters, the API server runs as a static pod.
You can add the flag to `/etc/kubernetes/manifests/kube-apiserver.yaml`.
Make sure the host directory `/etc/kubernetes/heptio-authenticator-aws/` is mounted into your API server pod.
You may also need to restart the kubelet daemon on your master node to pick up the updated static pod definition:
```
systemctl restart kubelet.service
```

### 4. Set up kubectl to use Heptio Authenticator for AWS tokens

> This requires a 1.10+ `kubectl` binary to work. If you receive `Please enter Username:` when trying to use `kubectl` you need to update to the latest `kubectl`

Finally, once the server is set up you'll want to authenticate!
You will still need a `kubeconfig` that has the public data about your cluster (cluster CA certificate, endpoint address).
The `users` section of your configuration, however, should include an exec section ([refer to the v1.10 docs](https://kubernetes.io/docs/admin/authentication/#client-go-credential-plugins))::
```yaml
# [...]
users:
- name: kubernetes-admin
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      command: heptio-authenticator-aws
      args:
        - "token"
        - "-i"
        - "CLUSTER_ID"
        - "-r"
        - "ROLE_ARN"
  # no client certificate/key needed here!
```

This means the `kubeconfig` is entirely public data and can be shared across all Authenticator users.
It may make sense to upload it to a trusted public location such as AWS S3.

Make sure you have the `heptio-authenticator-aws` binary installed.
You can install it with `go get -u -v github.com/heptio/authenticator/cmd/heptio-authenticator-aws`.

To authenticate, run `kubectl --kubeconfig /path/to/kubeconfig" [...]`.
kubectl will `exec` the `heptio-authenticator-aws` binary with the supplied params in your kubeconfig which will generate a token and pass it to the apiserver.
The token is valid for 15 minutes (the shortest value AWS permits) and can be reused multiple times.

You can also omit `-r ROLE_ARN` to sign the token with your existing credentials without assuming a dedicated role.
This is useful if you want to authenticate as an IAM user directly or if you want to authenticate using an EC2 instance role or a federated role.

## Kops Usage
Clusters managed by [Kops](https://github.com/kubernetes/kops) can be configured to use Authenticator.
Both single and HA master cluster configurations are supported.
Perform the following steps to setup Authenticator on a Kops cluster:
1. Pre-generate the certificate, key, and kubeconfig and upload them to the kops state store.
   ```
   heptio-authenticator-aws init -i $CLUSTER_NAME
   aws s3 cp cert.pem ${KOPS_STATE_STORE}/${CLUSTER_NAME}/addons/authenticator/cert.pem;
   aws s3 cp key.pem ${KOPS_STATE_STORE}/${CLUSTER_NAME}/addons/authenticator/key.pem;
   aws s3 cp heptio-authenticator-aws.kubeconfig ${KOPS_STATE_STORE}/${CLUSTER_NAME}/addons/authenticator/kubeconfig.yaml;
   ```
2. Add the following sections to the cluster spec, either using `kops edit cluster ${CLUSTER_NAME}` or editing the manifest yaml file.
   Be sure to replace `KOPS_STATE_STORE` and `CLUSTER_NAME` with their appropriate values since those environment variables are not available at runtime.
   This downloads the files from the state store on masters to a directory that is volume mounted by kube-apiserver.
   Kops does not support adding additional volumes to kube-apiserver so we must reuse the existing `/srv/kubernetes` hostPath volume.
   ```
   apiVersion: kops/v1alpha2
   kind: Cluster
   spec:
     kubeAPIServer:
       authenticationTokenWebhookConfigFile: /srv/kubernetes/heptio-authenticator-aws/kubeconfig.yaml
     hooks:
     - name: kops-hook-authenticator-config.service
       before:
         - kubelet.service
       roles: [Master]
       manifest: |
         [Unit]
         Description=Download Heptio AWS Authenticator configs from S3
         [Service]
         Type=oneshot
         ExecStart=/bin/mkdir -p /srv/kubernetes/heptio-authenticator-aws
         ExecStart=/usr/local/bin/aws s3 cp --recursive s3://KOPS_STATE_STORE/CLUSTER_NAME/addons/authenticator /srv/kubernetes/heptio-authenticator-aws/
   ```
  If using a non-default AMI that does not have the AWS CLI, replace the second ExecStart statement with

  ```
  ExecStart=/usr/bin/docker run --net=host --rm -v /srv/kubernetes/heptio-authenticator-aws:/srv/kubernetes/heptio-authenticator-aws quay.io/coreos/awscli@sha256:7b893bfb22ac582587798b011024f40871cd7424b9026595fd99c2b69492791d aws s3 cp --recursive s3://KOPS_STATE_STORE/CLUSTER_NAME/addons/authenticator /srv/kubernetes/heptio-authenticator-aws/
  ```
3. Apply the changes with `kops update cluster ${CLUSTER_NAME}`.
   If the cluster already exists, roll the cluster with `kops rolling-update cluster ${CLUSTER_NAME}` in order to recreate the master nodes.
4. Update the Authenticator DaemonSet's state and output volumes to both use `/srv/kubernetes/heptio-authenticator-aws/` for their `hostPath`s.
5. Apply the DaemonSet and ConfigMap resource manifests to launch the Authenticator server on the cluster.

*Note:* Certain Kops commands will overwrite the `ExecCredential` in kubeconfig so it may need to be restored manually. See [kubernetes/kops#5051](https://github.com/kubernetes/kops/issues/5051) for more information.


## How does it work?
It works using the AWS [`sts:GetCallerIdentity`](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetCallerIdentity.html) API endpoint.
This endpoint returns information about whatever AWS IAM credentials you use to connect to it.

#### Client side (`heptio-authenticator-aws token`)
We use this API in a somewhat unusual way by having the Heptio Authenticator for AWS client generate and pre-sign a request to the endpoint.
We serialize that request into a token that can pass through the Kubernetes authentication system.

#### Server side (`heptio-authenticator-aws server`)
The token is passed through the Kubernetes API server and into the Heptio Authenticator for AWS server's `/authenticate` endpoint via a webhook configuration.
The Heptio Authenticator for AWS server validates all the parameters of the pre-signed request to make sure nothing looks funny.
It then submits the request to the real `https://sts.amazonaws.com` server, which validates the client's HMAC signature and returns information about the user.
Now that the server knows the AWS identity of the client, it translates this identity into a Kubernetes user and groups via a simple static mapping.

This mechanism is borrowed with a few changes from [Vault](https://www.vaultproject.io/docs/auth/aws.html#iam-auth-method).

## What is a cluster ID?
The Authenticator cluster ID is a unique-per-cluster identifier that prevents certain replay attacks.
Specifically, it prevents one Authenticator server (e.g., in a dev environment) from using a client's token to authenticate to another Authenticator server in another cluster.

The cluster ID does need to be unique per-cluster, but it doesn't need to be a secret.
Some good choices are:
 - A random ID such as from `openssl rand 16 -hex`
 - The domain name of your Kubernetes API server

The [Vault documentation](https://www.vaultproject.io/docs/auth/aws.html#iam-auth-method) also explains this attack (see `X-Vault-AWS-IAM-Server-ID`).

## Specifying Credentials & Using AWS Profiles
Credentials can be specified for use with `heptio-authenticator-aws` via any of the methods available to the
[AWS SDK for Go](https://docs.aws.amazon.com/sdk-for-go/v1/developer-guide/configuring-sdk.html#specifying-credentials).
This includes specifying AWS credentials with enviroment variables or by utilizing a credentials file.

AWS [named profiles](https://docs.aws.amazon.com/cli/latest/userguide/cli-multiple-profiles.html) are supported by `heptio-authenticator-aws`
via the `AWS_PROFILE` environment variable. For example, to authenticate with credentials specified in the _dev_ profile the `AWS_PROFILE` can
be exported or specified explictly (e.g., `AWS_PROFILE=dev kubectl get all`). If no `AWS_PROFILE` is set, the _default_ profile is used.

The `AWS_PROFILE` can also be specified directly in the kubeconfig file
[as part of the `exec` flow](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#configuration). For example, to specify
that credentials from the _dev_ named profile should always be used by `heptio-authenticator-aws`, your kubeconfig would include an `env`
key thats sets the profile:

```yaml
apiVersion: v1
clusters:
- cluster:
    server: ${server}
    certificate-authority-data: ${cert}
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: aws
  name: aws
current-context: aws
kind: Config
preferences: {}
users:
- name: aws
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      command: heptio-authenticator-aws
      env:
      - name: "AWS_PROFILE"
        value: "dev"
      args:
        - "token"
        - "-i"
        - "mycluster"
```

This method allows the appropriate profile to be used implicitly. Note that any environment variables set as part of the `exec` flow will
take precedence over what's already set in your environment.

## Troubleshooting

If your client fails with an error like `could not get token: AccessDenied [...]`, you can try assuming the role with the AWS CLI directly:

```sh
# AWS CLI version of `heptio-authenticator-aws token -r arn:aws:iam::ACCOUNT:role/ROLE`:
$ aws sts assume-role --role-arn arn:aws:iam::ACCOUNT:role/ROLE --role-session-name test
```

If that fails, there are a few possible problems to check for:

 - Make sure your base AWS credentials are available in your shell (`aws sts get-caller-identity` can help troubleshoot this).

 - Make sure the target role allows your source account access (in the role trust policy).

 - Make sure your source principal (user/role/group) has an IAM policy that allows `sts:AssumeRole` for the target role.

 - Make sure you don't have any explicit deny policies attached to your user, group, or in AWS Organizations that would prevent the `sts:AssumeRole`.

## Full Configuration Format
The client and server have the same configuration format.
They can share the same exact configuration file, since there are no secrets stored in the configuration.

```yaml
# a unique-per-cluster identifier to prevent replay attacks (see above)
clusterID: my-dev-cluster.example.com

# default IAM role to assume for `heptio-authenticator-aws token`
defaultRole: arn:aws:iam::000000000000:role/KubernetesAdmin

# server listener configuration
server:
  # localhost port where the server will serve the /authenticate endpoint
  port: 21362 # (default)

  # state directory for generated TLS certificate and private keys
  stateDir: /var/heptio-authenticator-aws # (default)

  # output `path` where a generated webhook kubeconfig will be stored.
  generateKubeconfig: /etc/kubernetes/heptio-authenticator-aws.kubeconfig # (default)

  # role to assume before querying EC2 API in order to discover metadata like EC2 private DNS Name
  ec2DescribeInstancesRoleARN: arn:aws:iam::000000000000:role/DescribeInstancesRole

  # each mapRoles entry maps an IAM role to a username and set of groups
  # Each username and group can optionally contain template parameters:
  #  1) "{{AccountID}}" is the 12 digit AWS ID.
  #  2) "{{SessionName}}" is the role session name.
  mapRoles:
  # statically map arn:aws:iam::000000000000:role/KubernetesAdmin to cluster admin
  - roleARN: arn:aws:iam::000000000000:role/KubernetesAdmin
    username: kubernetes-admin
    groups:
    - system:masters

  # map EC2 instances in my "KubernetesNode" role to users like
  # "aws:000000000000:instance:i-0123456789abcdef0". Only use this if you
  # trust that the role can only be assumed by EC2 instances. If an IAM user
  # can assume this role directly (with sts:AssumeRole) they can control
  # SessionName.
  - roleARN: arn:aws:iam::000000000000:role/KubernetesNode
    username: aws:{{AccountID}}:instance:{{SessionName}}
    groups:
    - system:bootstrappers
    - aws:instances

  # map nodes that should conform to the username "system:node:<private-DNS>".  This
  # requires the authenticator to query the EC2 API in order to discover the private
  # DNS of the EC2 instance originating the authentication request.  Optionally, you
  # may specify a role that should be assumed before querying the EC2 API with the
  # key "server.ec2DescribeInstancesRoleARN" (see above).
  - roleARN: arn:aws:iam::000000000000:role/KubernetesNode
    username: system:node:{{EC2PrivateDNSName}}
    groups:
    - system:nodes
    - system:bootstrappers

  # map federated users in my "KubernetesAdmin" role to users like
  # "admin:alice-example.com". The SessionName is an arbitrary role name
  # like an e-mail address passed by the identity provider. Note that if this
  # role is assumed directly by an IAM User (not via federation), the user
  # can control the SessionName.
  - roleARN: arn:aws:iam::000000000000:role/KubernetesAdmin
    username: admin:{{SessionName}}
    groups:
    - system:masters

  # each mapUsers entry maps an IAM role to a static username and set of groups
  mapUsers:
  # map user IAM user Alice in 000000000000 to user "alice" in group "system:masters"
  - userARN: arn:aws:iam::000000000000:user/Alice
    username: alice
    groups:
    - system:masters

  # automatically map IAM ARN from these accounts to username.
  # NOTE: Always use quotes to avoid the account numbers being recognized as numbers
  # instead of strings by the yaml parser.
  mapAccounts:
  - "012345678901"
  - "456789012345"

```
