# Zero Trust Architecture Lab
## PlatformCon 2023

## TL;DR
This lab is a follow-up of the presentation blablabla at PLatformCon 2023


## Introduction

## Requirements
- **Docker/Podman**
- **Kubernetes**
This guide includes steps with [Kind](https://kind.sigs.k8s.io/) cluster.
- **nip.io**
To provide a valid DNS pointing to our localhost.
- **Local SSL Certificates**
This guide includes steps with [mkcert](https://github.com/FiloSottile/mkcert)
- **GitHub Account**
- **FIDO U2F Compatible Device**
It can be a physical Fido U2F Key (e.g. Yubico), a [virtual emulator](https://github.com/bulwarkid/virtual-fido) (Linux/Firefox), or a browser with native support (e.g. Chrome in macOS).
- Auth-chainer

## Auth-Chainer

## Installation steps
### Create a GitHub App

- Browse to [Settings > Developer settings > OAuth Apps > New Oauth App](https://github.com/settings/applications/new).

- Configure the following:
```
    Application name: ZTA Demo (or any other name)
    Homepage URL: https://private-7f000001.nip.io/
    Application description: Empty or any other text.
    Authorization callback URL: https://private-7f000001.nip.io/oauth2/callback
    Enable Device Flow: un-checked
```

- Once created, we need to **Generate a Client Secret**.
- Take note of the **Client ID** and the **Client Secret**.

### Install `mkcert` and provision local certificates
- Install [mkcert](https://github.com/FiloSottile/mkcert#installation)
- Install certificates

```

➜ mkcert -install

The local CA is already installed in the system trust store! 👍
The local CA is already installed in the Firefox and/or Chrome/Chromium trust store! 👍

➜ mkcert --cert-file kustomize/certificates/site-certificates.crt \
-key-file kustomize/certificates/site-certificates.key "*.nip.io"

Created a new certificate valid for the following names 📜
 - "*.nip.io"

Reminder: X.509 wildcards only go one level deep, so this won't match a.b.nip.io ℹ️

The certificate is at "kustomize/certificates/site-certificates.crt" and the key at "kustomize/certificates/site-certificates.key" ✅

It will expire on 7 August 2025 🗓
```

### Provision a Kubernetes & Ingress Nginx
[https://kind.sigs.k8s.io/docs/user/ingress/#ingress-nginx](https://kind.sigs.k8s.io/docs/user/ingress/#ingress-nginx)
```
➜  cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: zta-demo
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF

Creating cluster "zta-demo" ...
 ✓ Ensuring node image (kindest/node:v1.26.3) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-zta-demo"
You can now use your cluster with:

kubectl cluster-info --context kind-zta-demo

Have a nice day! 👋
```

```
➜ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

namespace/ingress-nginx created
serviceaccount/ingress-nginx created
serviceaccount/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
configmap/ingress-nginx-controller created
service/ingress-nginx-controller created
service/ingress-nginx-controller-admission created
deployment.apps/ingress-nginx-controller created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
ingressclass.networking.k8s.io/nginx created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created

➜  kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-528rm        0/1     Completed   0          30s
ingress-nginx-admission-patch-kr44d         0/1     Completed   2          30s
ingress-nginx-controller-6bfb7f5798-hcn5b   1/1     Running     0          30s
```

### Provision lab k8s resources

- Configure Oauth2-proxy
```
echo -n '[GITHUB_CLIENT_ID]' > kustomize/oauth2/client_id.secret
echo -n '[GITHUB_CLIENT_SECRET]' > kustomize/oauth2/client_secret.secret
```

Generate a secret with `dd if=/dev/urandom bs=32 count=1 status=none | base64 -w 0 | tr -- '+/' '-_'; echo` and edit the [cookie_secret.secret](./kustomize/oauth2/cookie_secret.secret) file.

Edit [oauth2-proxy.yaml](./kustomize/oauth2/oauth2-proxy.yaml) file to add your GitHub handler or to give access to a team ([info](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/overview/#environment-variables)).

- Configure WebAuthn proxy
```
cp kustomize/webauthn/webauth.credentials.yaml.example kustomize/webauthn/webauth.credentials.yaml
```
Generate a new cookie secret with the command the following command and edit the file [webauth.credentials.yaml](./kustomize/webauthn/webauth.credentials.yaml)

- Configure Auth-chainer

Generate a secret with `dd if=/dev/urandom bs=32 count=1 status=none | base64 -w 0 | tr -- '+/' '-_'; echo` and edit the [session_cookie_secret.secret](./kustomize/zta/session_cookie_secret.secret) file.

Example:
```
cookie_session_secrets:
  - ZILgrCRMYe3QHjr1bxo55pttS0lf6LQANNPOzh6XVA0=
user_credentials:
```


- Create the Kubernetes resources
```
➜ cd kustomize; kubectl -n zta-demo apply -k .

namespace/zta-demo created
configmap/echo-server-script-js-tmf4d5d5b4 created
secret/auth-chainer-256764bb4h created
secret/oauth2-proxy-f6b7mghmh6 created
secret/site-certificates-gg25mb4gt2 created
secret/webauthn-6kbmck2gg6 created
service/auth-chainer created
service/echo-server created
service/oauth2-proxy created
service/webauthn created
deployment.apps/auth-chainer created
deployment.apps/echo-server created
deployment.apps/oauth2-proxy created
deployment.apps/webauthn created
ingress.networking.k8s.io/auth-sites created
ingress.networking.k8s.io/private created
ingress.networking.k8s.io/public created

➜ kubectl get pods -n zta-demo
NAME                            READY   STATUS    RESTARTS   AGE
auth-chainer-85bff49d98-9snrm   1/1     Running   0          34s
echo-server-dbc4f7785-lf6vl     1/1     Running   0          34s
oauth2-proxy-c774549fd-nntdc    1/1     Running   0          34s
webauthn-5b9655748c-jl49t       1/1     Running   0          34s
```

## Usage
1. Browse to [https://private-7f000001.nip.io/](https://private-7f000001.nip.io/)
2. Authentificate using your GitHub credentials (remenber to configure correctly the allow GitHub principals).
3. Next, we'll be prompted in the WebAuthn (Device Authorisation) page. As our device is not currently authorised, the following:
3.1 Click on Register
![image](https://user-images.githubusercontent.com/13359249/236694775-cb276c76-4095-4075-9917-25d871e3f5a0.png)
3.2 If our Browser is compatible, a pop will appear. We need to accept it (this change depending of the browser/device)
3.3 We need to copy the output in [webauth.credentials.yaml](./kustomize/webauthn/webauth.credentials.yaml) file.
![image](https://user-images.githubusercontent.com/13359249/236694959-bfea338e-4655-4e10-b709-f3ede9561c5b.png)

Example:
```
cookie_session_secrets:
  - ZILgrCRMYe3QHjr1bxo55pttS0lf6LQANNPOzh6XVA0=
user_credentials:
  dlouvier: eyJJRCI6NjkxNDA2NjU5NTI1NzM4MDYxNiwiTmFtZSI6ImRsb3V2aWVyIiwiRGlzcGxheU5hbWUiOiJkbG91dmllciIsIkNyZWRlbnRpYWxzIjpbeyJJRCI6Im9nRllzTlJHTXNaNnlOVkVib0NzUXZnOFAvQmtsRWZYTkR5MG1lSG5vMjgrTTI2bGkva3JodWNlMDZDTDBGYmZpemUvZW5mRVdkL1lMRGIvYmxqQU5WbXBFN2ZjUi93SmdudHl4MFhzZ0N3eXNQYVFTZEFwRXpESnZmVWxQZFhSeFVMWmhFUFF6azdKbW9NUUtMcHJkcyt0ajJNaTRKZ3pmTTlwejBKR2hYeHVMVkY1bE9SelA4NVZjdUJINFE5SnM5OVVYcFlBQ2JXYUdDc21kdjFmbGMvSUhtblh1a2JLSTlzYWdIOXNxOFEzSmVqVUFreHI2ZTRDbUFLYjRvU1M0eUU9IiwiUHVibGljS2V5IjoicFFFQ0F5WWdBU0ZZSUhNL2VQcWMrcVJBRFBqaGdqYUMxNkRCUHI5cUU3bFJ5d0QyUnFrUHRwTW9JbGdnUnQvbUVBMGc5Q1EyS0xPcmN4YXpBNXVIbnlmUXJXcWU2c0NZNTR6WUQ3ND0iLCJBdHRlc3RhdGlvblR5cGUiOiJub25lIiwiVHJhbnNwb3J0IjpudWxsLCJGbGFncyI6eyJVc2VyUHJlc2VudCI6dHJ1ZSwiVXNlclZlcmlmaWVkIjpmYWxzZSwiQmFja3VwRWxpZ2libGUiOmZhbHNlLCJCYWNrdXBTdGF0ZSI6ZmFsc2V9LCJBdXRoZW50aWNhdG9yIjp7IkFBR1VJRCI6IkFBQUFBQUFBQUFBQUFBQUFBQUFBQUE9PSIsIlNpZ25Db3VudCI6MCwiQ2xvbmVXYXJuaW5nIjpmYWxzZSwiQXR0YWNobWVudCI6IiJ9fV19
```

4. Re-apply the kubernetes resources
`➜ cd kustomize; kubectl -n zta-demo apply -k .`

5. Browse again to [https://private-7f000001.nip.io/](https://private-7f000001.nip.io/) and now you will need to login.
6. The page will be opened.
