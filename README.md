# Configure vCluster with OIDC

This post follows on from my [blog post located here](https://vrelevant.net/vcluster-with-oidc/). See my other posts for more info on completely configuring the OIDC pieces.

We can configure a vCluster for OIDC auth at provisioning time. For this, I'm using the stock vCluster Helm chart. Refer to the blog post for background on the intent of this config.

To keep this page compact, I'm only showing the areas of the stock vCluster chart Values file you need to modify. As of this writing, the chart is using v1.26.1 K8s. I've changed it to 1.27.1 for my deployments. You'd need to update the image tags for the other control plane images defined in the chart as well.

Following from my post on OIDC with K8s, you should understand the purpose of `extraArgs` below. I'm then using a handy part of the Values file that lets us use variables within it (I think those are wrapped to tpl logic in the template). The manifestsTemplate Value will create the cluster-admin ClusterRoleBinding in the vCluster based on a value we supply at creation time.

Lastly, I'm creating a LoadBalancer service for the cluster so it will be accessible outside of the host cluster. This would ideally be an Ingress with wildcard URL match.

### Provision OIDC Enabled vCluster

1. Save a copy of this file to your local drive as `vals.yaml` and edit `--oidc-issuer-url` and `--oidc-client-id` to match your values:

```yaml
api:
  image: registry.k8s.io/kube-apiserver:v1.27.1
  extraArgs:
    - --oidc-issuer-url=https://dev-69716366.okta.com/oauth2/aus94zuko1QlEE5Qm5d7
    - --oidc-client-id=0oa94zu5m3tQpadSR5d7
    - --oidc-username-claim=email
    - --oidc-groups-claim=groups

init:
  manifestsTemplate: |-
    ---
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: oidc-cluster-admin
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
    - kind: Group
      name: {{ .Values.ClusterAdminGroup }}

service:
  type: LoadBalancer
```

2. Add the vCluster Helm repo 

```console
helm repo add loft-sh https://charts.loft.sh
helm repo update
```

3. Create the cluster

```console
kubectl create ns cluster-a
```

Replace the `team-a-cluster-admins` value below with a group from your Auth server. Members of this group will be bound to the cluster-admins role.

```console
helm install cluster-a loft-sh/vcluster-k8s -n cluster-a -f ./vals.yaml --set ClusterAdminGroup=team-a-cluster-admins
```

### Once the vCluster install is complete, we'll retrieve the generated kubeconfig file and modify it

1. Determine the LB External IP that was assigned and note for later

```console
kubectl get svc -n cluster-a cluster-a-lb
```

2. Retrieve kubeconfig file

```console
kubectl get secret vc-cluster-a -n cluster-a --template={{.data.config}} | base64 -D > kc
```

3. Open ./kc for editing

4. Delete every line below `users:`

It should look something like this:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJek1EUXlNREUyTURJME1Wb1hEVE16TURReE56RTJNREkwTVZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTGNCClJXcXp2NVpyRzMzeFZlbmR4b3NKeUtGc3k5SC94THBaYUpJM0lOZjQrMkY3bTZ2TWVTQm1BdmFuRFc1Wndzci8KQXZzaXhnWS9lSVFoV1NYTDcxdW9jcHllUldWQzdNc25CcjNuMVozRytpdnFhWXlhKzVGd0NrMmYxU1JNdWdlRwo4NDVUb1VmL1dqZy9MdGJCZ2lhck0rLzJCSFVYL0NoL2N3Qy9RdTltRTdnalk5a2RKM2YyUnBlT1ovdzF6dDZ3CkhVbUU2a01oQ00xSXBEZTMrNDNMc295TlA4WlcvM09sNVIxRlFFNG96V3BZckxJeGlycTd1MHZDME5aSFVkQzUKRXBkWUNDZVNVbStWM0MxdWRMV1R2eFl2RVZNNlRpL21jbHhvQ1l6cjd5WHU2dzZuUlZWMlJXMGxQejZCZng5RApHK2V5RFpaWGhHcjhZK1dmUWtNQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZBU05mTlpBOENUK1k4YXV1MXRSeUlBMVZSc2dNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBSjZQb1psN3BLN3hUK0VnSTM5aQpjNURrVjViaFFpSGpKdG5hT3lmcm5pV3pNVHBVQ1I1S3lzU0t3MmdOMUk4dFZKMStUYWNYdktOeUIyL2hjQXlPClNuNjZ2NTdoSGlQd0dsTzZqbkRpcE5lM1BndTVkUnFxT2V3Zldzd25TeTFoZWpWc2NsZ1BBQmhyWldQSmlYYW8KUDRKa0cwSEpRNXdJVmlMcW5Xd2JkY2dtaGd5aHZobFJKUk9CMU5rZkNzUTNWZnJXYU9nMncvck45aGJXOHhmcApJNnp5em5vWkpjcVZ2RFVMWG5SSTBQd1hMK2tNMnVmalhSM3BJM3BPeTJzdlltZVBUOGlaK0JqYjJ1ZkNnUnJGCmhSbk5wbHFPRGJXa01tTlBEOWZVYlIxd1hNZXJQaHIrVEQveHVmZ2VhZUxRdlhuQ2s2S1NMZmNIb0dlWVVCR1oKWVZVPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://cluster-a-api:443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
```

5. Replace host name in `server:`url with LB IP you noted earlier (e.g. `server: https://192.168.253.2:443`)

6. Search and replace all occurreces of the words `kubernetes` and `kubernetes-admin` with `cluster-a` _(This kubeconfig file will not store any specific user identity, it only points to our cluster and directs us to OIDC Auth to prove identity. So we do not need references to users or roles within it)_

7. Add the following under `Users:` and edit OIDC issuer and client to match yours

```yaml
- name: cluster-a
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: kubectl
      args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=https://dev-69716366.okta.com/oauth2/aus94zuko1QlEE5Qm5d7
      - --oidc-client-id=0oa94zu5m3tQpadSR5d7
      - --oidc-extra-scope="email offline_access profile openid"
```

Once complete, you should have a kubeconfig that looks something like below. You can post this file on a billboard if you'd like to:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJek1EUXlNREUyTURJME1Wb1hEVE16TURReE56RTJNREkwTVZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTGNCClJXcXp2NVpyRzMzeFZlbmR4b3NKeUtGc3k5SC94THBaYUpJM0lOZjQrMkY3bTZ2TWVTQm1BdmFuRFc1Wndzci8KQXZzaXhnWS9lSVFoV1NYTDcxdW9jcHllUldWQzdNc25CcjNuMVozRytpdnFhWXlhKzVGd0NrMmYxU1JNdWdlRwo4NDVUb1VmL1dqZy9MdGJCZ2lhck0rLzJCSFVYL0NoL2N3Qy9RdTltRTdnalk5a2RKM2YyUnBlT1ovdzF6dDZ3CkhVbUU2a01oQ00xSXBEZTMrNDNMc295TlA4WlcvM09sNVIxRlFFNG96V3BZckxJeGlycTd1MHZDME5aSFVkQzUKRXBkWUNDZVNVbStWM0MxdWRMV1R2eFl2RVZNNlRpL21jbHhvQ1l6cjd5WHU2dzZuUlZWMlJXMGxQejZCZng5RApHK2V5RFpaWGhHcjhZK1dmUWtNQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZBU05mTlpBOENUK1k4YXV1MXRSeUlBMVZSc2dNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBSjZQb1psN3BLN3hUK0VnSTM5aQpjNURrVjViaFFpSGpKdG5hT3lmcm5pV3pNVHBVQ1I1S3lzU0t3MmdOMUk4dFZKMStUYWNYdktOeUIyL2hjQXlPClNuNjZ2NTdoSGlQd0dsTzZqbkRpcE5lM1BndTVkUnFxT2V3Zldzd25TeTFoZWpWc2NsZ1BBQmhyWldQSmlYYW8KUDRKa0cwSEpRNXdJVmlMcW5Xd2JkY2dtaGd5aHZobFJKUk9CMU5rZkNzUTNWZnJXYU9nMncvck45aGJXOHhmcApJNnp5em5vWkpjcVZ2RFVMWG5SSTBQd1hMK2tNMnVmalhSM3BJM3BPeTJzdlltZVBUOGlaK0JqYjJ1ZkNnUnJGCmhSbk5wbHFPRGJXa01tTlBEOWZVYlIxd1hNZXJQaHIrVEQveHVmZ2VhZUxRdlhuQ2s2S1NMZmNIb0dlWVVCR1oKWVZVPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://192.168.253.2:443
  name: cluster-a
contexts:
- context:
    cluster: cluster-a
    user: cluster-a
  name: cluster-a@cluster-a
current-context: cluster-a@cluster-a
kind: Config
preferences: {}
users:
- name: cluster-a
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: kubectl
      args:
      - oidc-login
      - get-token
      - --oidc-issuer-url=https://dev-69716366.okta.com/oauth2/aus94zuko1QlEE5Qm5d7
      - --oidc-client-id=0oa94zu5m3tQpadSR5d7
      - --oidc-extra-scope="email offline_access profile openid"
 ```
 
 8. Save your new kubeconfig file to your local drive as `kc`
 
 ### Test authentication to your vCluster
 
 Login with a user that is a member of the group you specified in step 3 of creating the vCluster.
 
 ```console
 kubectl get ns --kubeconfig=./kc
 ```
 
 If everything is configured correctly, you will be redirected to authenticate with your Auth server and then be granted admin access to the vCluster instance. With this foundation, we can build out the automation further for a smoother flow.
