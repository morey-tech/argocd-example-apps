# Creating a fully-managed Kubernetes GitOps platform with Argo CD

Argo CD version: v2.5.1
Kubernertes Version: v1.25.2

## Lab
### Intro
what is: Kubernetes, GitOps, Argo CD, Akuity.

### Fork this repo.
The lab will use this repo and files in it as the environment configuration for the Kubernetes cluster.

`https://github.com/morey-tech/argocd-example-apps`

### Create a Kubernetes cluster locally.
1. Run `docker run hello-world` to check that docker is running and that you have access. You should the following:
   ```
   Hello from Docker!
   This message shows that your installation appears to be working correctly.
   ```
2. Run `kubectl version` to check that it is installed.
   ```
   Client Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.2", GitCommit:"5835544ca568b757a8ecae5c153f317e5736700e", GitTreeState:"clean", BuildDate:"2022-09-21T14:33:49Z", GoVersion:"go1.19.1", Compiler:"gc", Platform:"darwin/arm64"}
   Kustomize Version: vXX.XX.XX
   ```
   - You may also see a message indicating that it couldn't connect to the server to retrive it's version. This happens if your kubectl context is unset. It defaults to connecting to a cluster on localhost which will fail if a cluster is not present. This is okay for now since, we will be creating the cluster later in this section.
      ```
      The connection to the server localhost:8080 was refused - did you specify the right host or port?
      ```
3. Run `kind version` to check that it is installed.
   ```
   kind v0.16.0 go1.19.1 darwin/arm64
   ```
4. Copy the `kind-cluster.yaml` from the repo fork to a local file.
5. Run `kind create cluster --config <path/to/config> --name <cluster-name>` to create the cluster.
   ```
   export CONFIG=./kind-cluster.yaml
   export CLUSTER_NAME=morey-tech
   kind create cluster --config $CONFIG --name $CLUSTER_NAME
   ```
   ```
   Creating cluster "morey-tech" ...
   ✓ Ensuring node image (kindest/node:v1.25.2) 🖼
   ✓ Preparing nodes 📦 📦 📦 📦  
   ✓ Writing configuration 📜 
   ✓ Starting control-plane 🕹️ 
   ✓ Installing CNI 🔌 
   ✓ Installing StorageClass 💾 
   ✓ Joining worker nodes 🚜 
   Set kubectl context to "kind-morey-tech"
   You can now use your cluster with:

   kubectl cluster-info --context kind-morey-tech

   Thanks for using kind! 😊
   ```
6. Check that the cluster is working by running `kubectl get nodes`.
   ```
   % kubectl get nodes
   NAME                            STATUS   ROLES           AGE   VERSION
   morey-tech-control-plane   Ready    control-plane   74s   v1.25.2
   morey-tech-worker          Ready    <none>          54s   v1.25.2
   morey-tech-worker2         Ready    <none>          54s   v1.25.2
   morey-tech-worker3         Ready    <none>          54s   v1.25.2
   ```
### Launch an Argo CD instance.
1. Create an account on the Akuity Platform.
2. Create an organization.

3. On the sidebar, click "Argo CD".
4. In the top right, click "Create".
5. Set the "Instance Name" to `cluster-manager`.
6. Select the Version correisponding to `v2.5`.
7. Click "Create".
8. For the Argo CD instance, click "Settings".
9. In the "General" section, find "Declarative Management" and enable it by clicking the toggle.
10. In the top right, click "Save".
11. On the inner sidebar, under "Security & Access", click "System Accounts".
12. Enable the "Admin Account" by clicking the toggle, and clicking "Confirm"
13. Then for the `admin` user, click "Set Password".
14. To get the password, click "Copy".
   - If the copy option does not appear, click "Regenerate Password" and then "Copy". Note, this will invalidate any other sessions for the `admin` user.
15. In the bottom right of the Set password prompt, hit "Close"
16. In the top, next to the Argo CD instance name and status, click the instance URL.
17. Log into the Argo CD UI with the username `admin` and the password copied earlier

### Deploy an agent to the cluster.
1. Back on the Akuity Platform in the top left of the dashboard for the Argo CD instance, click "Clusters".
2. Click "Connect a cluster" in the Clusters dashboard for the Argo CD instance.
3. Enter the "Cluster Name".
4. In the bottom right, Click "Connect Cluster".
5. To get the agent install command, click "Copy to Clipboard" then, in the bottom right, "Done".
6. Open your terminal and ensure that `kubectl` is connected to the correct cluster. `kubectl config current-context`
7. Paste and run the command against the cluster.
8. Check the pods in the `akuity` namespace. Wait for the `Running` status on all of the pods (apprx. 1 minute).
   ```
   % kubectl get pods -n akuity
   NAME                                                        READY   STATUS    RESTARTS   AGE
   akuity-agent-<replicaset-id>-<pod-id>                       1/1     Running   0          65s
   akuity-agent-<replicaset-id>-<pod-id>                       1/1     Running   0          65s
   argocd-application-controller-<replicaset-id>-<pod-id>      2/2     Running   0          65s
   argocd-notifications-controller-<replicaset-id>-<pod-id>    1/1     Running   0          65s
   argocd-redis-<replicaset-id>-<pod-id>                       1/1     Running   0          65s
   argocd-repo-server-<replicaset-id>-<pod-id>                 1/1     Running   0          64s
   argocd-repo-server-<replicaset-id>-<pod-id>                 1/1     Running   0          64s
   ```
9.  Confirm healthy in the Clusters dashboard.

### Create the App of Apps in Argo CD
1. Navigate back to the Argo CD UI, and click "NEW APP".
2. In the top right, click "EDIT AS YAML".
3. Paste the contents of `app-of-apps.yaml` in the root of the repo.
4. Update `metadata.name` to match your cluster name.
5. Update `spec.source.repoURL` to match your repo name.
6. Click "SAVE"
7. Review settings in Wizard translated from the YAML, then click "CREATE".
8. Click on the card for the Application.
9. In the top bar, click "SYNC" then "SYNCHRONIZE". This will instruct Argo CD to create the resources defined by the Application. Afterwards, all resources in the tree will show a green checkmark. Indicating that they are present in the cluster.

### Enable auto-sync for the `app-namespaces` Application.
1. On the `app-namespaces` Application resource, click the link button (arrow pointing out of the top right of a square). This will open the Application view for it.
2.  In the top menu, click "APP DETAILS".
3.  Under the "SYNC POLICY" section, click "ENABLE AUTO-SYNC". This will cause the application to immediately create the missing resources for the Application (i.e., the namespaces).
   1. You can see this by clicking out of the App Details page and looking for the green checkmark on the resources.
4.  To the right of the "SELF HEAL", click "ENABLE". This will ensure that if a resource is deleted in the cluster (i.e., drift from the desired state) that Argo CD will automatically recreate it. This is important for namespaces as they are required for the Application using that namespace.
   2. We will not enable "PRUNE RESOURCES" because the Application is managing namespace resources, who's deletion could be catostraophic. If you are confident in the processes surrounding changes to your environment configuratin repo, then this could be enabled allowing for automatic clean-up.
5.  To demonstrate the auto-heal functionality, from your terminal use `kubectl` to delete the `helm-guestbook` namespace.
   ```
   % kubectl delete ns/helm-guestbook
   namespace "helm-guestbook" deleted
   ```
   - You'll notice that the namespace has been recreated by Argo CD (see the recent age compared to the other namespaces).

### Enable auto-sync to Application. Make change to it via Git.

### [bonus] Delete the cluster, recreate it, and deploy the agent to bootstrap it.
 - This could also demonstrate how the set up could easily be replicated in a hosted K8s environment with no additional configuration for the Applications or the firewall of the hosted environment.
 - The control plane allows for the cluster to be bootstrapped by simply deploying the Akuity agent.
