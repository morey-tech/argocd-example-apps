# Creating a fully-managed Kubernetes GitOps platform with Argo CD

In this hour-long hands-on lab, we will set up a repo to contain the environment configuration, create a Kubernetes cluster, and provision an Argo CD instance. Then use Argo CD to deploy Applications from the repo to the cluster. Finally, we will demonstrate the auto-sync and auto-heal functionality of the Applications.

Ultimately, you will have a Kubernetes cluster, with Applications deployed using an Argo CD control plane.

Argo CD version: v2.5.1
Kubernertes version: v1.25.2
kind version: v0.16.0

- [Creating a fully-managed Kubernetes GitOps platform with Argo CD](#creating-a-fully-managed-kubernetes-gitops-platform-with-argo-cd)
  - [Lab](#lab)
    - [Intro](#intro)
    - [Pre-requisetes](#pre-requisetes)
    - [1. Fork this repo.](#1-fork-this-repo)
    - [2. Create a Kubernetes cluster using `kind`.](#2-create-a-kubernetes-cluster-using-kind)
    - [2. Launch an Argo CD instance.](#2-launch-an-argo-cd-instance)
    - [3. Deploy an agent to the cluster.](#3-deploy-an-agent-to-the-cluster)
    - [4. Create an Application in Argo CD.](#4-create-an-application-in-argo-cd)
    - [5. Change the target revision for the Application.](#5-change-the-target-revision-for-the-application)
    - [6. Enable auto-sync for the helm-guestbook Application.](#6-enable-auto-sync-for-the-helm-guestbook-application)
    - [7. Demonstrate Application auto-sync via Git.](#7-demonstrate-application-auto-sync-via-git)
    - [8. Create an App-of-Apps in Argo CD.](#8-create-an-app-of-apps-in-argo-cd)
    - [9. Add another Application to the App of Apps.](#9-add-another-application-to-the-app-of-apps)
    - [[bonus] Delete the cluster, recreate it, and deploy the agent to bootstrap it.](#bonus-delete-the-cluster-recreate-it-and-deploy-the-agent-to-bootstrap-it)

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

Anywhere you see text in the format `<...>`, this indicates that you need to replace it with the value relevant to your scenario. Using my scenario for example, in `https://github.com/<github-username>/argocd-example-apps`, I would replace `<github-username>` with `morey-tech`.

### 1. Fork this repo.
The lab will use this repo and files in it as the environment configuration for the Kubernetes cluster (i.e., the "GitOps repo").

You will start by forking this repo to your Github account. 
1. Click [this link](https://github.com/morey-tech/argocd-example-apps/fork).
2. Ensure the desired "Owner" is selected (e.g., your personal account and not an org).
3. Then click "Create fork".

<!-- Need to remove argocd-example-apps repo as parent -->

Once you have a fork of the repo, create a new branch to contain the changes made during this lab. This will later be referred to as the "feature branch" (e.g., `<feature-branch>`). For simplicity, name it using the current date.
1. Navigate to `https://github.com/<github-username>/argocd-example-apps/branches`
2. In the top right, click "New branch".
3. Set the "Branch name" to the current date (e.g. `2022-11-08`).
4. In the bottom right of the panel, click "Create branch".

### 2. Create a Kubernetes cluster using `kind`.
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
This is were the "fully-managed" part comes in. The Akuity Platform provides Argo CD as a service. By spinning up an instace on Akuity, you'll have an external Argo CD control plane that can manage clusters in private networks. 

Just like how GitHub is hosting the manifests for your apps, the Akuity Platform host, not only Argo CD, but all of the Application, ApplicationSet, and AppProject definitions. This solves the problem of needing a Kubernetes cluster to host Argo CD so that it can manage other clusters.

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

    At this point, your Argo CD instance will begin intializing. This typically takes under 2 minutes.

    To get the instance ready for the rest of the lab, there are a couple of steps left.

12. In the dashboard for the Argo CD instance, click "Settings".
13. In the "General" section, find "Declarative Management" and enable it by clicking the toggle.
    
    This enables using the Akuity Platform with the App of Apps pattern or ApplicationSets. We will demonstrate this later in the lab.

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
Now that you have an Argo CD instance running and assecible, you need to connect your cluster to it. With the Akuity Platform, you will deploy an agent to the cluster enabling it to connect it to the control plane.

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
An Application declaritively tells Argo CD what Kubernetes recoures to deploy. In this section, you will create one to deploy the `helm-guestbook` Helm Chart, from your GitOps repo, to your cluster.

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
    
    This manifest describes an Application. The name of the Application is `helm-guestbook`. The source is the GitOps repo with the Helm chart. The destination is the cluster conntected to the Argo CD control plane. The sync policy will automatically create the namespace.

4. Update `spec.destination.name` to match your environment name (i.e., cluster name).
5. Update `spec.source.repoURL` to match your repo name.
6. Click "SAVE".
   1. At this point, the UI has translated the Application manifest into the corriesponding fields in the wizard.
7. In the top left, click "CREATE".

   This will close the new app pane and show the card for the Application you created. The status on the card will show "Missing" and "OutOfSync".

8. Click on the Application card titled `argocd/helm-guestbook`.
   1. In this state, the Application resource tree shows what was generated from the source repo URL and path defined. Because auto-sync is disable, the resources does not exist in the destination yet.
   2. Click "APP DIFF" to see what manifests the Application rendered.
9.  In the top bar, click "SYNC" then "SYNCHRONIZE". This will instruct Argo CD to create the resources defined by the Application.

The tree will expand as the Deployment creates a ReplicaSet that then creates a pod, and the Service creates an Endpoint and EndpointSlice.

Afterwards, all resources in the tree will show a green checkmark. Indicating that they are present in the cluster.

### 5. Change the target revision for the Application.
Currently the Application is tracking the `HEAD` of the repo (in this case the `main` branch). To facilitate testing of changes on feature branches, the source target revision can be changed. You will update this to track the branch you created after forking the repo.

1.  In the top menu, click "APP DETAILS".
2.  In the top right, click "EDIT".
3.  Update the "TARGET REVISION" to the branch created after forking the repo.
4.  In the top right, click "SAVE".
5.  In the top right of the App Details pane, click the X to close it.

Now, the Application will watch for any changes made to the `helm-guestbook` Helm chart on your feature branch.

### 6. Enable auto-sync for the helm-guestbook Application.
You can automate the deployment of changes to your feature branch by enabling the automated sync policy. This will cause any change to the Application source in the repo or Application itself, to be applied with intervention.

1.  In the top menu, click "APP DETAILS".
2.  Under the "SYNC POLICY" section, click "ENABLE AUTO-SYNC".
3.  In the top right of the App Details pane, click the X to close it.

If the Application was out-of-sync, this would immediately trigger a sync. In this case, your Application is already in sync, so no changes are made.

The default sync interval is 3 minutes. Meaning, any changes made in Git may not apply for up to 3 minutes.

### 7. Demonstrate Application auto-sync via Git.
With auto-sync enabled on the `helm-guestbook` Application, you can now make a change in Git to have it applied automatically in your cluster.

1. Navigate to the `helm-guestbook/values.yaml`.
2. In the top right of the file, click the pencil icon to edit.
3. Update the `image.tag` to the `0.2` list.
4. A commit message. For example `chore: bump tag to 0.2 for helm-guestbook`.
5. In the bottom left, click "Commit changes".
6. Switch back to the Argo CD UI and go to the `argocd/helm-guestbook` Application.
7. In the top right, click the "REFRESH" button.
   1. This will trigger Argo CD to check for any changes to the Application source and resources. This would happen automatically on a 3 minute (default) interval.

Due to the change in the repo, Argo CD will detect that the Application is out-of-sync. It will template the Hlme chart, do a three-way diff, and patch the `helm-guestbook` deployment with the new image tag, triggering a rolling update.

### 8. Create an App-of-Apps in Argo CD.

https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern

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

At this point, the Application will be out-of-sync. The diff will show the addition of the `argocd.argoproj.io/tracking-id` label to the existing `helm-guestbook` Application. This indicates that it's now managed by the `morey-tech` Application (e.g., App-of-Apps).

The 

```diff
  kind: Application
  metadata:
++  annotations:
++    argocd.argoproj.io/tracking-id: 'morey-tech:argoproj.io/Application:argocd/helm-guestbook'
++  finalizers:
++    - resources-finalizer.argocd.argoproj.io
    generation: 44
    labels:
      ...
      path: helm-guestbook
      repoURL: 'https://github.com/morey-tech/argocd-example-apps'
    syncPolicy:
      automated: {}
```

We are using the `targetRevision` of the Parent Application as a Helm parameter, to set the `targetRevision` of the child Applications (rendered by the Helm chart). When the `targetRevision` is changed on the parent Application, it will template the Helm chart with the new value. The child Applications will become out-of-sync due to their new `targetRevision`.

### 9. Add another Application to the App of Apps.
Now that the App of Apps has been deployed and is managing the existing `helm-guestbook` Application, you can deploy a second Application simply by adding the path to the values in the Helm chart.

1. Navigate to `file and line`.
2. In the top right of the file, click the pencil icon to edit.
3. Add `kustomize-guestbook` to the `applications` list.
4. A commit message. For example `chore: deploy kustomize-guestbook to <environment-name>`.
5. In the bottom left, click "Commit changes".
6. Switch back to the Argo CD UI.
7.  The CURRENT SYNC STATUS will show "OutOfSync" indicating that it is out-of-sync.
    1.  If the status still shows synced, click on the downward arrow of the "REFRESH" button and then click "Hard Refresh".
    2.  The Application and new Application resource will show a yellow circle with a white arrow.

### [bonus] Delete the cluster, recreate it, and deploy the agent to bootstrap it.
 - This could also demonstrate how the set up could easily be replicated in a hosted K8s environment with no additional configuration for the Applications or the firewall of the hosted environment.
 - The control plane allows for the cluster to be bootstrapped by simply deploying the Akuity agent.