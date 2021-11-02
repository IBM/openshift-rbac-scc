# OpenShift RBAC with ServiceAccounts and SecurityContextConstraints

When discussing container application security on OpenShift there are multiple layers that you should to take into consideration. According to [Red Hat](https://www.redhat.com/en/resources/layered-approach-security-detail), the layers to be considered include:

- Building security into applications - Using secure and trusted container images, using private image registries
- Manage configuration, security, and compliance of applications - **Identity and access management**, platform configuration, **policy based deployments**
- Protecting running applications - Ensuring container isolation, application and network isolation

In this lab we will be exploring **identity and access managment** by creating a service account with limited permissions and **policy based deployments** by creating a custom SecurityContextConstraint that a pod must abide by in order to be deployed.

This lab is comprised of the following sections:

1. [Setup](#Setup)
1. [Section 1: Authorization and User Permissions](#Section-1:-Authorization-and-User-Permissions)
1. [Section 2: Container Permissions and SCCs](#Section-2:-Container-Permissions-and-SCCs)

## Setup

The following must be done before you can get started on the lab.

1. Gain access to your lab environment by following the steps found [here](https://ibm.github.io/workshop-setup/GRANTCLUSTER/){:target="_blank"}

1. As mentioned in the link above, you can use the IBM Cloud Shell for the workshop which can be accessed here: [https://cloud.ibm.com/shell](https://cloud.ibm.com/shell){:target="_blank"}

1. Once in the terminal, clone the workshop repository by entering:

    ```sh
    git clone https://github.com/IBM/openshift-rbac-scc.git
    cd openshift-rbac-scc/src
    ```

## Section 1: Authorization and User Permissions

### Introduction

Users in OpenShift are only allowed to perform actions that are defined in the **Roles** or **ClusterRoles** that they are assigned. **Roles** can be thought of as a list of rules in the form of verbs ("get", "create", "list", "delete", etc.) and the resources that they apply to (pods, projects, deployments, users, etc.) within a single project. **ClusterRoles** are the same concept as **Roles** but are available in all projects.

A **RoleBinding** ties a **Role** or **ClusterRole** to a user to a single project while a **ClusterRoleBinding** ties a **ClusterRole** to a user and is available across all projects.

The same permissions process mentioned above with **Roles** and **RoleBindings** for a user is also true for **ServiceAccounts**. **ServiceAccounts** are accounts in a cluster that can view and manage resources through the Kubernetes API without being tied to users. This makes **ServiceAccounts** ideal for applications or operators that need to view or manage cluster resources.

This diagram from the [OpenShift documentation](https://docs.openshift.com/container-platform/4.5/authentication/using-rbac.html) summarizes RBAC policies:

![RBACDiagram](../images/RBACDiagram.png)

In this first section of the lab we will be creating a `helper` **ServiceAccount** that can be used to monitor cluster resources in project named `rbac-project`. We will be going through the following steps:

- Creating a **ServiceAccount**
- Viewing built-in **ClusterRoles**
- Creating a **RoleBinding** between the `view` **ClusterRole** and the `helper` **ServiceAccount**
- Authenticating as the **ServiceAccount** and testing permissions

### Steps

1. Log into your cluster by following the [steps outlined here](https://ibm.github.io/workshop-setup/ROKS/#login-to-openshift){:target="_blank"}.

1. It is recommended that you create a new project when deploying applications rather than working in the `default` project so let's create one.

    ```sh
    oc new-project rbac-project
    ```

    You will automatically target the new project with subsequent `oc` commands.
    
    A project is a namespace with additional annotations to isolate their resources.
    
    ```sh
    $ oc describe project rbac-project
    Name:             rbac-project
    Created:          19 minutes ago
    Labels:           <none>
    Annotations:      openshift.io/description=
                      openshift.io/display-name=
                      openshift.io/requester=IAM#user@email.com
                      openshift.io/sa.scc.mcs=s0:c25,c15
                      openshift.io/sa.scc.supplemental-groups=1000630000/10000
                      openshift.io/sa.scc.uid-range=1000630000/10000
    Display Name:     <none>
    Description:      <none>
    Status:           Active
    Node Selector:    <none>
    Quota:            <none>
    Resource limits:  <none>
    ```
    
    OpenShift allocated 3 ranges to the project: mcs, supplemental-groups, and uid-range. 

    * sa.scc.mcs - Multi-Category Security (MCS) are SELinux labels that represent the owner of the process and is comprised of 3 unique sub-values sX:cY,cZ for each process-owner. MCS labels are used to distinguish between containers in different namespaces.
    * sa.scc.supplemental-groups - Group ID or GID range, Supplemental-Groups are used for shared storage (NFS/GlusterFS), fsGroup for block storage (Ceph RBD, iSCSI, etc),
    * sa.scc.uid-range - UID, UID range.

1. Each new project that is created contains 3 Service Accounts. Let's take a look at them:

    ```sh
    oc get sa
    ```

    You should see three listed:

    ```sh
    NAME        SECRETS   AGE
    builder     2         77m
    default     2         77m
    deployer    2         77m
    ```

    Each service account has it's own purpose:

    * builder: Used when running `build` pods
    * deployer: Used when running `deploy` pods
    * default: Default service account used if none is specified in the pod template

    We will actually see an example of a build pod later but it is typically used with the `oc new-app` and `oc start-build` commands.

1. Create a service account

    ```sh
    oc create sa helper
    ```

    In our fake example, this service account will be used to view the resources that are running on our cluster by querying the underlying Kubernetes API.

1. View the service accounts again:

    ```sh
    oc get sa
    ```

    You should now see:

    ```sh
    NAME       SECRETS   AGE
    builder    2         133m
    default    2         133m
    deployer   2         133m
    helper     2         132m
    ```

1. Let's authenticate with the API to see what permissions our new Service Account has:

    1. Switch to your terminal and paste in `export TOKEN=` without entering yet.
    1. Navigate back to your OpenShift Console that should be open in another browser tab.

    1. Click on `User Management` to expand the menu then click on `Service Accounts`

        ![service accounts](../images/serviceAccounts.png)

    1. In the service accounts page, select `rbac-project` from the `Project` dropdown near the top of the page above the heading `Service Accounts`.

    1. In the list of service accounts, click on `helper`

        ![helper](../images/helper.png)

    1. Scroll down to the area labeled `Secrets` and click on the secret that starts with `helper-token`.

        ![helperTokenName](../images/helperTokenName.png)

    1. On the secret page, scroll down and find the data field labeled `token`
    1. Click on the copy icon on the right of the `token` field.

        ![helperToken](../images/helperToken.png)

    1. Switch back to your terminal and paste in the token. It should be a really long string of characters. It will look something like:

        ```sh
        export TOKEN=eyJhbGciOiJSUzI1NiIsImtpZCI6ImJrOUYzcUhSaGEwMk9mMXZMcWppWVpuZWdCQjBjcVM1d3ZJRk13ODluTGsifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJyYmFjLXByb2plY3QiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlY3JldC5uYW1lIjoiaGVscGVyLXRva2VuLXo5eDJ3Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImhlbHBlciIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImM4M2RiMDFiLWRiOTAtNGI2NS1iZWVkLWU4YzU2ZTM2ODkwNiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpyYmFjLXByb2plY3Q6aGVscGVyIn0.JFw8vgrgVJhjAuP_8L5JVkPFPfK2w11Rzevi56qPlYvpUPaW4XJdVv3y3_8ieUWNGkWALO25--oj8Bqb8NYQ8mJuvh1D2vRL743iwJ8-fLah9KJO0wXe-OKNWNZAizSuz2EH3uJHpP6FqVJ8I-a0h015caS5VtrA16dSmShTer1i1JpSO8NxtUgnZLQtzfnkaclKyvvIFF8xcby83r8SWZGyGB7mxH7jQ5zdyw1cLwBgIXVXuSOpJA_4JmLMo3CZ2q8H-Nq20_3mD548z-fvv9vUxwgGSLTR1WGTPb2XDC5pEt3vF0oZULez406PHmF5Hd5fdLV2XGWXE1fHnnNoEA
        ```

    1. Then press enter

    Now that we have the token we need the server address of the cluster that we are authenticating with.

    1. In your terminal paste in `export SERVER=` without entering yet.
    1. Open up the `Copy login command` tab again and copy the server address.

        ![serverAddress](../images/server.png)

    1. Go back to your terminal and paste in the server address so that the entire command reads something like:

        ```sh
        export SERVER=https://c104-e.us-east.containers.cloud.ibm.com:30166
        ```

        Then press enter.

    1. Now let's authenticate to the cluster as our `helper` service account:

        ```sh
        oc login --server=$SERVER --token=$TOKEN
        ```

1. As the new service account, try retrieving a list of all projects

    ```sh
    oc get projects
    ```

1. How about getting all resources?

    ```sh
    oc get all
    ```

    You should see a bunch of errors regarding resources that this service account does not have access to. This is good as it means our permissions are working as intended.

1. Let's switch back to our own user account. Copy the login command from the `Copy login command` tab again.

1. To give our new service account some permissions we will assign it the ClusterRole of `view`. This will allow `helper` to view the resources on the cluster without being able to modify anything. 

    First take a look at what permissions exactly we are giving `helper`:

    ```sh
    oc project rbac-project
    oc describe clusterrole view
    ```

    This looks like the ideal list of permissions for `helper` so let's grant him the ClusterRole.

    ```sh
    oc adm policy add-cluster-role-to-user view -z helper
    ```

1. Now let's deploy something for `helper` to view.

    ```bash
    oc new-app ruby~https://github.com/sclorg/ruby-ex.git
    ```

    This will create a simple ruby application for us. Remember when I mentioned that we would see build pods? Here is where they come into play. A build pod will build the sample app using the `builder` service account and will stop running when it is done and the application pods will then be available. There is a lot to go into regarding OpenShift builds but since that isn't the focus of this lab, we will leave that for another time.

1. Go to your terminal and login as `helper` again.

    ```sh
    oc login --server=$SERVER --token=$TOKEN
    ```

1. Now try running the following commands:

    ```sh
    oc get projects
    ```

    ```sh
    oc get pods -n rbac-project
    ```

    If our permissions are correct, `helper` can view the pods we just spun up with our user account.

    What if we try deploying something as `helper`?

    ```sh
    oc new-app ruby~https://github.com/sclorg/ruby-ex.git
    ```

    Notice in the `Creating Resources section` of the output that every resource failed to create due to the permissions that we have set.

In summary, we create a new service account for a `helper` that would be able to monitor workloads on our cluster and we assigned a limited ClusterRole of `view` that would ensure that it only has read access to our resources.

## Section 2: Container Permissions and SCCs

### Introduction

In the previous section we discussed permissions using RBAC with **Roles** and **RoleBindings** to restrict what a user or **ServiceAccount** could do and now in this section we will take a look at using RBAC policies with **SecurityContextConstraints** (SCC) to restrict what a pod can do. 

**SecurityContextConstraints** restrict what a pod has access to and in order for a pod to be deployed, it must abide by the conditions outlined in the SCC. OpenShift comes with a few SCCs built in such as:

- anyuid
- hostaccess
- hostmount-anyuid
- hostnetwork 
- node-exporter
- nonroot
- privileged
- restricted (default)

In OpenShift, a deployed pod will run under the `default` **ServiceAccount** unless a specific **ServiceAccountName** is listed in the pod template. The pod uses the SCC of the attached **ServiceAccount** to determine if the pod can be deployed. In this case, `default` uses the `restricted` SCC which is the most locked-down SCC.

The `restricted` SCC should work for most workloads but if you need to create your own, this lab will show you how.

For more on SCCs, check out the [OpenShift Docs](https://docs.openshift.com/container-platform/4.5/authentication/managing-security-context-constraints.html ).

We will be building upon concepts that were covered in the previous section so I recommend going through Section 1 first.

In this section we will:

- Examine the `restricted` SCC
- Create a custom SCC
- Create a **ServiceAccount** that we will add the SCC to
- Create a **Role** that allows use of the custom SCC
- Create a **RoleBinding** between the new **Role** and **ServiceAccount**
- Test out the new restrictions

### Steps

1. Let's log back into our own user account. Copy the login command from the `Copy Login Command` tab again.

1. Create a new project for this section

    ```sh
    oc new-project scc-project
    ```

1. Examine the default `restricted` scc.

    ```sh
    oc describe scc restricted
    ```

    Notice how many of the permissions under `Settings` are disabled such as `Allow Privileged`, `Allow Host Network`, `Allow Host PID`, and more. 
    
    One of the few things not restricted in this list is `Read Only Root Filesystem`. This setting will make the running pod completely immutable and unable to be written to at all. Usually, permissions in the Linux OS within the pod will prevent the user from writing to the important directories, but if you really want a secure locked down pod you can set `Read Only Root Filesystem` to true and make the pod completely read-only.

    Before we create a custom read-only SCC let's see what a pod looks like running with the default `restricted` SCC.

1. Create a deployment with the default SCC.

    ```sh
    oc create -f RHELDeploy.yaml
    ```

1. Find the pod name that was just deployed

    ```sh
    oc get pods
    ```

1. Then, exec into the pod by running the following while replacing `{pod name}` with the actual pod name from the previous command:

    ```sh
    oc exec -it {pod name} -- bash
    ```

1. Once inside the pod, try the following:

    ```sh
    touch test.txt
    ```

    This should fail as RHEL has a lot of security in place to ensure that users do not have access to most locations.

    Now try this:

    ```sh
    touch tmp/test.txt
    ```

    This should work.

    Let's lock down the image even more by making the image read-only.

    Exit the pod by entering `exit`

1. Delete the deployment

    ```sh
    oc delete -f RHELDeploy.yaml
    ```

1. Let's create our own custom SCC

    ```sh
    oc create -f readonly-scc.yaml
    ```

    Let's examine the new SCC and see how it compares to `restricted`

    ```sh
    oc describe scc read-only
    ```

    It's very similar to `restricted` and the only major difference being that `Read Only Root Filesystem` is set to true now.

1. Create Service account

    ```sh
    oc create sa read-only
    ```

1. Now, create a Role that allows us to use the SCC with a service account

    ```sh
    oc create -f readonly-role.yaml
    ```

1. Then create a RoleBinding that ties the `read-only` service account to the new role that was just created.

    ```sh
	oc create -f readonly-rolebinding.yaml
    ```

1. Before we try deploying the RHEL pod again we first need to specify in our deployment manifest (RHELDeploy.yaml) that we want to use our `read-only` service account to run the pod.

    If you feel comfortable editing the file using `vi` you can edit the RHELDeploy.yaml file to remove the `#` on line 17. If not, here are some alternatives that you can copy and paste in the terminal:

    This command depends on what operating system you are working on.

    If on Linux (such as on IBM Cloud Shell) try:
    ```sh
	sed -i 's/#//' RHELDeploy.yaml
    ```

    If on MacOSx try:

    ```sh
    sed -i '' 's/#//' RHELDeploy.yaml
    ```

1. With the comment removed, we can now deploy the file.

    First, let's take a look at the RHELDeploy.yaml file

    ```sh
    cat RHELDeploy.yaml
    ```

    Examine the contents of the file and notice the `serviceAccountName:` field. This ensures that this service account will be used.

    Deploy the application with:

    ```sh
	oc create -f RHELDeploy.yaml
    ```

1. Let's go ahead and exec into our pod like before and see what has changed

    1. Get all pods
	    
        ```sh
        oc get pods
        ```

    1. Then, exec into the pod by running the following while replacing `{pod name}` with the actual pod name from the previous command:

        ```sh
        oc exec -it {pod name} -- bash
        ```

	1. Once inside the pod, try a few commands to test out the permissions now:
    
        Try creating a file in the root directory

        ```sh
        touch test.txt
        ```

        That command still fails but now it should have a different message regarding operating as a "read-only file system"

        Let's try creating a file in the "/tmp" directory now. This worked last time we tried with the default permissions.

	    ```sh
        touch tmp/test.txt
        ```

        This time the command doesn't work and we get the same error as the previous command about running in a "read-only file system".

        What this means is that the running pod is now completely immutable; it cannot be modified maliciously. There is a caveat, however. Some applications actually rely on being able to write to the local file system. The workaround to this problem requires mounting a volume to write to instead.

## Summary
In summary, this lab walked you through two different scenarios: using RBAC to create a helper service account to monitor workloads through the Kubernetes API, and using RBAC to ensure that service accounts can use custom SCCs to run pods.
