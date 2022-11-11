# Creating a fully-managed Kubernetes GitOps platform with Argo CD

In this hour-long hands-on lab, we will set up a repo to contain the environment configuration, create a Kubernetes cluster, and provision an Argo CD instance. Then use Argo CD to deploy Applications from the repo to the cluster. Finally, we will demonstrate the auto-sync and auto-heal functionality of the Applications.

Ultimately, you will have a Kubernetes cluster, with Applications deployed using an Argo CD control plane.

Argo CD version: v2.5.1
Kubernertes version: v1.25.2
kind version: v0.16.0

## Lab
### Intro
Use these links to learn what is: [Kubernetes](https://youtu.be/4ht22ReBjno?t=164), [GitOps](https://opengitops.dev/), [Argo CD](https://argo-cd.readthedocs.io/en/stable/#what-is-argo-cd), [Akuity](https://akuity.io/).

### Pre-requisetes
The lab requires that you have:
- a [GitHub](https://github.com/) Account - This will be used to create the repo for the environment configuration and for creating an account on the [Akuity Platform](https://akuity.io/akuity-platform/).
- the Kubernetes command-line tool, [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) - Will be used for interacting with the cluster directly.
- a Kubernetes cluster with Cluster Admin access (to create namespaces and cluster roles) or;
- [Docker Desktop](https://duckduckgo.com/?q=docker+desktop+install) and [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) installed on your local machine.
- [egress traffic](https://www.webopedia.com/definitions/egress-traffic/) allowed from the Kubernetes cluster to the internet.

To prepare for the lab scenario, you can think of what to name your *organization* and *environment*. For example, my *organization* could be named “[ACME Co.](https://en.wikipedia.org/wiki/Acme_Corporation)” and my *environment* “southwest”.
- The *organization* name will be used for setting up your account on the Akuity Platform.
- The *environment* name will be used for the cluster and the Application in Argo CD.

Anywhere you see text in the format `<...>`, this indicates that you need to replace it with the value relevant to your scenario. Using my scenario for example:
- In `https://github.com/<github-username>/argocd-example-apps`, I would replace `<github-username>` with `morey-tech`.
- 

### 1. Fork this repo.
The lab will use this repo and files in it as the environment configuration for the Kubernetes cluster (i.e., the "GitOps repo").

You will start by forking this repo to your Github account. 
1. Click [this link](https://github.com/morey-tech/argocd-example-apps/fork).
2. Ensure the desired "Owner" is selected (e.g., your personal account and not an org).
3. Then click "Create fork".

<!-- Need to remove argocd-example-apps repo as parent -->

Once you have a fork of the repo, create a new branch to contain the changes made during this lab. For simplicity, name it using the current date.
1. Navigate to `https://github.com/<github-username>/argocd-example-apps/branches`
2. In the top right, click "New branch".
3. Set the "Branch name" to the current date (e.g. `2022-11-08`).
4. In the bottom right of the panel, click "Create branch".

### 2. Create a Kubernetes cluster locally.
If you brought your own Kubernetes cluster to the lab, you can skip this section.

Otherwise, you will be creating one on your local mahcine using `kind`.

1. Run `docker run hello-world` to check that Docker is running and that you have access. You should the following:
   ```
   Hello from Docker!
   This message shows that your installation appears to be working correctly.
   ```
2. Run `kubectl version` to check that `kubectl` is installed.
   ```
   Client Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.2", GitCommit:"5835544ca568b757a8ecae5c153f317e5736700e", GitTreeState:"clean", BuildDate:"2022-09-21T14:33:49Z", GoVersion:"go1.19.1", Compiler:"gc", Platform:"darwin/arm64"}
   Kustomize Version: vXX.XX.XX
   ```
   - You may also see a message indicating that it couldn't connect to the server to retrive it's version. This happens if your kubectl context is unset, defaulting to connecting to a cluster on localhost which will fail if a cluster is not present, or if your context is set to a cluster that is not available.

    This is okay for now since, we will be creating the cluster later in this section.
      ```
      The connection to the server localhost:8080 was refused - did you specify the right host or port?
      ```
3. Run `kind version` to check that `kind` is installed.
   ```
   kind v0.16.0 go1.19.1 darwin/arm64
   ```
4. Copy the `kind-cluster.yaml` from the repo fork to a local file.
   ```
   curl https://raw.githubusercontent.com/<github-username>/argocd-example-apps/main/kind-cluster.yaml > kind-cluster.yaml
   ```
5. Run the following commands to create the cluster.
   ```
   export CONFIG=./kind-cluster.yaml
   export CLUSTER_NAME=morey-tech
   kind create cluster --config $CONFIG --name $CLUSTER_NAME
   # or
   CLUSTER_NAME=morey-tech kind create cluster --config ./kind-cluster.yaml --name $CLUSTER_NAME

   kind create cluster --config ./kind-cluster.yaml --name <environment-name>
   ```
   You should see the follow output:
   ```
   Creating cluster "<environment-name>" ...
   ✓ Ensuring node image (kindest/node:v1.25.2) 🖼
   ✓ Preparing nodes 📦 📦 📦 📦  
   ✓ Writing configuration 📜 
   ✓ Starting control-plane 🕹️ 
   ✓ Installing CNI 🔌 
   ✓ Installing StorageClass 💾 
   ✓ Joining worker nodes 🚜 
   Set kubectl context to "kind-<environment-name>"
   You can now use your cluster with:

   kubectl cluster-info --context kind-<environment-name>

   Thanks for using kind! 😊
   ```
6. Check that the cluster is working by running `kubectl get nodes`.
   ```
   % kubectl get nodes
   NAME                               STATUS   ROLES           AGE   VERSION
   <environment-name>-control-plane   Ready    control-plane   74s   v1.25.2
   <environment-name>-worker          Ready    <none>          54s   v1.25.2
   <environment-name>-worker2         Ready    <none>          54s   v1.25.2
   <environment-name>-worker3         Ready    <none>          54s   v1.25.2
   ```
    - Does kind always update the kubectl context? Maybe check that too

### 2. Launch an Argo CD instance.
1. Create an account on the [Akuity Platform](https://akuity.io/signup).
2. To log in with GitHub SSO, click "Continute with GitHub".
   1. You can also use Google SSO or a email and password combo. For the sake of the lab, I'm assuming you will be using GitHub.
3. Click "Authorize akuityio".
4. Create an organization by clicking "create or join" in the information bulletin.
5. In the top right, click "New organization".
6. Enter your "Organization Name" and click "Create"

7. Near the top of the sidebar, click "Argo CD".
8. In the top right, click "Create".
9. Set the "Instance Name" to `cluster-manager`.
10. Under the "Version" section, click the option correisponding to `v2.5`.
11. Click "Create".
12. In the dashboard for the Argo CD instance, click "Settings".
13. In the "General" section, find "Declarative Management" and enable it by clicking the toggle.
    1.  This enables using the Akuity Platform to host the Application CRs generated when using the App of Apps pattern or ApplicationSets. We will demonstrate this later in the lab.
14. In the top right, click "Save".
15. On the inner sidebar, under "Security & Access", click "System Accounts".
16. Enable the "Admin Account" by clicking the toggle, and clicking "Confirm" on the prompt.
17. Then for the `admin` user, click "Set Password".
18. To get the password, click "Copy".
   - If the copy option does not appear, click "Regenerate Password" and then "Copy". Note, this will invalidate any other sessions for the `admin` user.
15. In the bottom right of the Set password prompt, hit "Close"
16. In the top, next to the Argo CD instance name and status, click the instance URL (e.g., `<uuid>.cd.akuity.cloud`). This will open the Argo CD login page.
17.  Enter the username `admin` and the password copied previously.

You will be presented with the Argo CD Applications dashboard.

### 3. Deploy an agent to the cluster.
1. Back on the Akuity Platform, in the top left of the dashboard for the Argo CD instance, click "Clusters".
2. In the top right, click "Connect a cluster".
3. Enter your *environment* name as the "Cluster Name".
4. In the bottom right, Click "Connect Cluster".
5. To get the agent install command, click "Copy to Clipboard". Then, in the bottom right, "Done".
6. Open your terminal and ensure that `kubectl` is connected to the correct cluster. 
    ```
    % kubectl config current-context
    kind-<env-name>
    ```
    - This output assumes you are using `kind`.
7. Paste and run the command against the cluster.
   1. This will create the `akuity` namespace and deploy the resources for the Akuity Agent.
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
   - Re-run the `get pods` command to check for updates on the pod statuses.
9.  Back on the Clusters dashboard, confirm that the cluster shows a green heart before the name, indicating a healthy status.

### 4. Create an Application in Argo CD.
1. Navigate back to the Argo CD UI, and click "NEW APP".
2. In the top right, click "EDIT AS YAML".
3. Paste the contents of `helm-guestbook.yaml` (in the root of the repo).
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: helm-guestbook
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: 'https://github.com/<github-username>/argocd-example-apps' # Update to match your fork.
        path: helm-guestbook
        targetRevision: HEAD
      destination:
        namespace: helm-guestbook
        name: <environment-name> # Update this value.
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
    ```
4. Update `spec.destination.name` to match your environment name (i.e., cluster name).
5. Update `spec.source.repoURL` to match your repo name.
6. Click "SAVE"
7. Review the settings translated from YAML in Wizard. Then, in the top left, click "CREATE".
8. Click on the Application card titled `argocd/helm-guestbook`.
   1. In this state, the Application resource tree shows what was generated from the source repo URL and path defined. Because auto-sync is disable, the resources does not exist in the destination yet.
   2. Click "APP DIFF" to see what manifests the Application rendered.
9.  In the top bar, click "SYNC" then "SYNCHRONIZE". This will instruct Argo CD to create the resources defined by the Application.

Afterwards, all resources in the tree will show a green checkmark. Indicating that they are present in the cluster.

### 5. Enable auto-sync for the helm-guestbook Application.
To automate the deployment of changes to the parent Application, you will enable the automated sync policy. This will cause any change to the Application source in the repo or Application itself, to be applied immediately.

1.  In the top menu, click "APP DETAILS".
2.  Under the "SYNC POLICY" section, click "ENABLE AUTO-SYNC".
    1.  If the Application was out-of-sync, this would immediately trigger a sync.
3.  In the top right of the App Details pane, click the X to close it.

To demonstrate the auto-sync functionality, you will update the `image.tag` Helm parameter via the Argo CD UI.

1.  In the top menu, click "APP DETAILS".
2.  Go to the "PARAMETERS" tab and in the top right, click "EDIT"
3.  Set the "image.tag" parameter to `0.2`. Then, in the top right, click "SAVE".
    1.  This will cause the application to sync and do a rolling upgrade of the deployment.
4.  In the top right of the App Details pane, click the X to close it.

You will see the Application progressing and a new pod staring up with the `0.2` image tag. Then the old pod running `0.1` will be deleted.

### 6. Create an App-of-Apps in Argo CD.
1. Navigate back to the Argo CD UI, and click "NEW APP".
2. In the top right, click "EDIT AS YAML".
3. Paste the contents of `app-of-apps.yaml` (in the root of the repo).
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: <environment-name> # Update to your cluster name.
      namespace: argocd
      finalizers:
      - resources-finalizer.argocd.argoproj.io
    spec:
      destination:
        name: in-cluster
      project: default
      source:
        path: apps
        repoURL: https://github.com/<github-username>/argocd-example-apps # Update to your repo URL.
        targetRevision: HEAD
        helm:
          # Use the build enviroment provided by Argo CD to inherit the values used to
          # template the child Applications.
          parameters:
          - name: spec.destination.name
            value: $ARGOCD_APP_NAME
          - name: spec.source.repoURL
            value: $ARGOCD_APP_SOURCE_REPO_URL
          - name: spec.source.targetRevision
            value: $ARGOCD_APP_SOURCE_TARGET_REVISION
    ```
4. Update `metadata.name` to match your environment name (i.e., cluster name).
5. Update `spec.source.repoURL` to match your repo name.
6. Click "SAVE"
7. Review the settings translated from YAML in Wizard. Then, in the top left, click "CREATE".
8. Click on the Application card titled `argocd/<environment-name>`.

At this point, the Application will be out-of-sync, with a diff showing that it will add a label to the existing `helm-guestbook` Application to indicate that it's now managed by the App-of-Apps.

### 7. Change the target revision for the App-of-Apps.
Currently the Application is tracking the `HEAD` of the repo. You will update this to track the branch created after forking the repo. This will cause your cluster's environment to reflect your feature branch instead of `HEAD`.

<!-- 1. On the `<environment-name>` Application resource, click the link button (arrow pointing out of the top right of a square). This will open the Application view for it. -->
1.  In the top menu, click "APP DETAILS".
2.  In the top right, click "EDIT".
3.  Update the "TARGET REVISION" to the branch created after forking the repo.
4.  In the top right, click "SAVE".
5.  In the top right of the App Details pane, click the X to close it.
6.  The CURRENT SYNC STATUS will show "OutOfSync" indicating that it is out-of-sync.
    1.  The Application and Application resources will show a yellow circle with a white arrow.
- You can see the diff by going to the top left corner and clicking "APP DIFF".
  ```diff
        path: helm-guestbook
      repoURL: 'https://github.com/<your-username>/argocd-example-apps'
  --  targetRevision: HEAD
  ++  targetRevision: '<your-feature-branch>'
  status:
    health:
  ```

We are using the `targetRevision` of the Parent Application as a Helm parameter, to set the `targetRevision` of the child Applications (rendered by the Helm chart). When the `targetRevision` is changed on the parent Application, it will template the Helm chart with the new value. The child Applications will become out-of-sync due to their new `targetRevision`.
   <!-- 1. You can see this by clicking out of the App Details page and looking for the green checkmark on the resources. -->
<!-- 3.  To the right of the "SELF HEAL", click "ENABLE". This will ensure that if a resource is deleted in the cluster (i.e., drift from the desired state) that Argo CD will automatically recreate it. This is important for namespaces as they are required for the Application using that namespace. -->
   <!-- 2. We will not enable "PRUNE RESOURCES" because the Application is managing other Application resources, who's deletion could be *catostraophic*. If you are confident in the processes surrounding changes to your environment configuratin repo, then this could be enabled allowing for automatic clean-up. -->
<!-- 6.  To demonstrate the auto-heal functionality, from your terminal use `kubectl` to delete the `helm-guestbook` namespace.
   ```
   % kubectl delete ns/helm-guestbook
   namespace "helm-guestbook" deleted
   ```
   - You'll notice that the namespace has been recreated by Argo CD (see the recent age compared to the other namespaces). -->

### Demonstrate Application auto-sync via Git.
1. Navigate to `file and line`.
2. In the top right of the file, click the pencil icon to edit.
3. Add `kustomize-guestbook` to the `applications` list.
4. A commit message. For example `chore: deploy kustomize-guestbook to <environment-name>`.
5. In the bottom left, click "Commit changes".
6. Switch back to the Argo CD UI.

### [bonus] Delete the cluster, recreate it, and deploy the agent to bootstrap it.
 - This could also demonstrate how the set up could easily be replicated in a hosted K8s environment with no additional configuration for the Applications or the firewall of the hosted environment.
 - The control plane allows for the cluster to be bootstrapped by simply deploying the Akuity agent.