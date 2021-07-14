<h1>Continuous Integration and Continuous Delivery (CICD) on OpenShift - Reference Implementation </h1>

<br/>

This article will attempt to walk you through a Reference Implementation of OpenShift Continuous Integration and Continuous Delivery (CICD) using a sample Quarkus project. For the sake of simpicity, the article will be broken down into 2 sections. They are as follows:-

<Ul>
    <li><a href="#ci">Continuous Integration Reference Implementation on OpenShift using OpenShift Pipelines (Tekton Project)</a></li>
    <li><a href="#cd">Continuous Delivery Reference Implementation on OpenShift using OpenShift GitOps (ArgoCD)</a></li>
</Ul>

<br/>

<h2 id="ci">Continuous Integration Reference Implementation on OpenShift using OpenShift Pipelines (Tekton Project)</h2>

<br/>

OpenShift Pipelines are based on the Tekton project (https://tekton.dev) - a new, Kubernetes and container native way to manage pipelines.

The purpose of this article is demonstrate a reference implementation for the Openshift Pipelines using a sample Quarkus project.

The pipeline will look like this:-
<br/>

<img src="ci-pipeline-dev.png"></img>

<br/>

The pipeline will:
<Ul>
    <li>Clone the source code repository (https://gitlab.xxx.corp.xxx.ca/AI/aiocp/sample-reactive-quarkus-app.git) on commit to <code>dev</code> branch</li>
    <li>Run test cases</li>
    <li>Build artifact for image</li>
    <li>In parallel scan build artifact for vulnerabilities using a <a href="https://github.com/aquasecurity/trivy"><code>Trivy</code></a> task </li>
    <li>Build an tag image using the source code <code>commitId</code></li>
    <li>In parallel scan image locally for vulnerabilities using a <a href="https://github.com/aquasecurity/trivy"><code>Trivy</code></a> task</li>
    <li>Push image to image repository - artifactory.xxxx.corp.xxx.ca:5073 (<code>ccop-dev</code> repo)</li>
    <li>Scan image in remote image repository for vulnerabilities using a <a href="https://github.com/aquasecurity/trivy"><code>Trivy</code></a> task</li>
    <li>Update the repository https://gitlab.xxx.corp.xxx.ca/AI/aiocp/tekton-pipeline.git under the path <code>k8s/overlays/dev</code> in <code>dev</code> branch to point to the latest image in the Artifactory registry</li> 
</Ul>

<br/>
The Goal of this article is:
<Ul>
    <li>Understand how to setup a CICD pipeline using OpenShift pipelines(Tekton) and OpenShift GitOps(ArgoCD)</li>
    <li>Serve as a quickstart for your CICD pipeline</li>
</Ul>


<h3>Prerequisites</h3>
<br/>

The following are required to run this reference pipeline (and possibly your pipeline)

<Ul>
    <li>Ask your Cluster Administrator to install <b>Red Hat OpenShift Pipelines</b> incase you can't find it in the <b>Installed Operators</b> <img src="ci-operator.png"></img></li>
    <li>Create an Openshift namespace - (In this case - <code>cop-pipeline</code>)</li>
    <li>Create or get access to the reference source code repository (https://gitlab.xxx.corp.bce.ca/AI/aiocp/sample-reactive-quarkus-app.git)</li>
    <li>Create or get access to the reference k8s repository for continuous deployment (https://gitlab.xxx.corp.bce.ca/AI/aiocp/tekton-pipeline.git)</li>
    <li>From your profile page, in the source code repository kindly setup an Access Token with the right permissions as shown in the screenshot below:-<img src="pat.png"></img>
    (Kindly note that the same Access Token was used for the k8s repository in the reference implementation. If you have different user profiles, you will have to setup multiple Access tokens)</li>
    <li>Setup a secret for the Access Token(s) in the Openshift namespace (<code>cop-pipeline</code>) and annotate the secret(s) (<code>tekton.dev/git-0: 'https://gitlab.xxx.corp.xxx.ca'</code>) as shown in the yaml file below:-
    <pre>
    <code>
    apiVersion: v1
    metadata:
        name: gitlab-token
        namespace: cop-pipeline
    annotations:
        tekton.dev/git-0: 'https://gitlab.xxx.corp.xxx.ca'
    data:
        password: xxxxxxxxxxxxxxxxxx=
        username: xxxxxx==
    type: kubernetes.io/basic-auth
    </code>
    </pre>
    </li>
    <li>Add the secrets into the service account <code>pipeline</code> as shown in the yaml file below:-
     <pre>
    <code>
    kind: ServiceAccount
    apiVersion: v1
    metadata:
        name: pipeline
        namespace: cop-pipeline
    secrets:
        - name: pipeline-token-xxxxx
        - name: pipeline-dockercfg-xxxx
        - name: gitlab-token
        - name: artifactory-token
    imagePullSecrets:
        - name: pipeline-dockercfg-xxxxx
    </code>
    </pre>
    </li>
</Ul>
<br/>
<h3>Run CI Pipeline on Dev Environment</h3>
<p>You can either manually run the pipeline or create <a href="#wh">webhooks on Gitlab to trigger the pipeline run. Kindly refer to this for a source yaml file for the pipeline - <a href="https://gitlab.xxx.corp.xxx.ca/AI/aiocp/tekton-pipeline/-/blob/master/pipelines/ci-pipeline.yaml">ci-pipeline.yaml</a> </p>
<br/>
<p>The screenshot below shows a Successful Pipeline Run on the Dev Environment</p>
<img src="pipeline.png"></img>

<p>The screenshot below shows a Successful Pipeline Run on the Dev Environment</p>
<img src="ci-dev-pipeline-failed.png"></img>

<br/>
<h3 id="#wh">Creating webhooks on Gitlab to trigger the pipeline run</h3>
<p>On OpenShift, kindly setup the following Triggers</p>
<ul>
<li>Create a Trigger Template with the yaml file below:-
<pre>
<code>
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerTemplate
metadata:
  name: ci-pipeline-template
  namespace: cop-pipeline
spec:
  params:
    - name: git-repo-url
    - name: git-revision
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: ci-pipeline-
      spec:
        params:
          - name: git-source-url
            value: $(tt.params.git-repo-url)
          - name: git-source-revision
            value: $(tt.params.git-revision)
          - name: SONAR_URL
            value: 'http://sonarqube:9000'
          - name: SONAR_AUTH_TOKEN
            value: xxxxxxxxxxxxxxxxxxxxx
          - name: LOCAL_SCAN_PATH
            value: ./
          - name: LOCAL_IMAGE_SCAN_PATH
            value: ./
          - name: REMOTE_IMAGE_URL
            value: >-
              image-registry.openshift-image-registry.svc:5000/cop-pipeline/xxx-app-image
          - name: SEVERITY_LEVELS
            value: CRITICAL
          - name: KUSTOMIZE_GIT_URL
            value: 'https://gitlab.xxx.corp.xxx.ca/AI/aiocp/tekton-pipeline.git'
          - name: KUSTOMIZE_GIT_CONTEXT_DIR
            value: k8s/overlays/dev
        pipelineRef:
          name: ci-pipeline
        workspaces:
          - name: app-source
            persistentVolumeClaim:
              claimName: workspace-pvc2
          - name: maven-settings
            persistentVolumeClaim:
              claimName: maven-settings-pvc
          - name: shared-image-repo
            persistentVolumeClaim:
              claimName: workspace-pvc2
          - emptyDir: {}
            name: kustomize-repo

</code>
</pre>
</li>

<li>Create a Trigger Binding with the yaml file below:-
<pre>
<code>
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerBinding
metadata:
  name: ci-pipeline-binding
  namespace: cop-pipeline
spec:
  params:
    - name: git-repo-url
      value: $(body.repository.git_http_url)
    - name: git-revision
      value: $(body.after)
</code>
</pre>
</li>

<li>Create an Event listener with the yaml file below:-
<pre>
<code>
apiVersion: triggers.tekton.dev/v1alpha1
kind: EventListener
metadata:
  name: ci-pipeline
  namespace: cop-pipeline
spec:
  namespaceSelector: {}
  podTemplate: {}
  resources: {}
  serviceAccountName: pipeline
  triggers:
    - bindings:
        - kind: TriggerBinding
          ref: ci-pipeline-binding
      name: ci-pipeline
      template:
        ref: ci-pipeline-template
</code>
</pre>
</li>

<li><p>On the Gitlab source code repository (https://gitlab.xxx.corp.xxx.ca/AI/aiocp/sample-reactive-quarkus-app.git), go to <code>Settings/webhooks</code> and setup a webhook for the repo to initiate a Pipeline Run on commit. Kindly see screenshot below:-</p>
<p><img src="webhook.png"></img></p></li>
</ul>
<br/>

<h3>Promoting to UAT Environment</h3>
<p>Kindly refer to this file for the yaml source - <a href="https://gitlab.xxx.corp.xxx.ca/AI/aiocp/tekton-pipeline/-/blob/master/pipelines/uat-cd-pipeline.yaml">uat-cd-pipeline.yaml</a></p>
<p>The pipeline will look like this:-</p>
<img src="uat-prod-pipeline.png"></img>
<p>The pipeline will:
    <ul>
        <li>Skopeo copy from dev image repository to UAT image repository</li>
        <li>Scan image in remote image repository for vulnerabilities using a <a href="https://github.com/aquasecurity/trivy"><code>Trivy</code></a> task</li>
        <li>Update the repository https://gitlab.xxx.corp.xxx.ca/AI/aiocp/tekton-pipeline.git under the path <code>k8s/overlays/uat</code> in <code>uat</code> branch to point to the latest image in the Artifactory registry</li> 
    </ul>
</p>
<br/>

<h3>Promoting to Prod Environment</h3>
<p>Kindly refer to this file for the yaml source - <a href="https://gitlab.xxx.corp.xxx.ca/AI/aiocp/tekton-pipeline/-/blob/master/pipelines/prod-cd-pipeline.yaml">prod-cd-pipeline.yaml</a></p>
<p>The pipeline will look like this:-</p>
<img src="uat-prod-pipeline.png"></img>
<p>The pipeline will:
    <ul>
        <li>Skopeo copy from dev image repository to UAT image repository</li>
        <li>Scan image in remote image repository for vulnerabilities using a <a href="https://github.com/aquasecurity/trivy"><code>Trivy</code></a> task</li>
        <li>Update the repository https://gitlab.xxx.corp.xxx.ca/AI/aiocp/tekton-pipeline.git under the path <code>k8s/overlays/prod</code> in <code>master</code> branch to point to the latest image in the Artifactory registry</li> 
    </ul>
</p>
<br/>
<br/>

<h2 id="cd">Continuous Delivery Reference Implementation on OpenShift using OpenShift GitOps (ArgoCD)</h2>
<br/>
<p>Ask your Cluster Administrator to install <b>Red Hat OpenShift GitOps</b> incase you can't find it in the <b>Installed Operators</b> <img src="gitops-operator.png"></img> </p>
<br/>
<h3>Setting up a Tenant ArgoCD instance</h3>
<p>The following are steps for setting up a tenant ArgoCD instance</p>
<ul>
<li>Create an argocd instance namespace on OpenShift</li>
<li>Create all the namespaces that the argocd instance will manage i.e namepaces to deploy the applications. This is typically done through the Frontdoor (https://openshift.int.xxx.ca). If you have access to create namespaces, you can use the code snippet below:
<pre>
<code>
for namespace in ccop-ref-dev ccop-ref-uat ccop-ref-prod
do
    oc new-project $namespace
done;
</code>
</pre>
</li>

<li>
Create a cluster config secret in the argocd instance namespace
<pre>
<code>
apiVersion: v1
stringData:
  config: '{"tlsClientConfig":{"insecure":false}}'
  name: in-cluster
  namespaces: ccop-ref-dev,ccop-ref-uat,ccop-ref-prod
  server: https://kubernetes.default.svc
kind: Secret
metadata:
  annotations:
    managed-by: argocd.argoproj.io
  labels:
    argocd.argoproj.io/secret-type: cluster
  name: in-cluster
type: Opaque
</code>
</pre>
</li>

<li>
 Create an argocd instance in the <argocd instance namespace>
<pre>
<code>
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: argocd
  namespace: <argocd instance namespace>
spec:
  server:
    route:
      enabled: true
</code>
</pre>
</li>

<li>
 Ask the cluster-admin to create a local <code>cluster-admin</code> rolebinding in all the managed namespaces configured in step 2 for the application controller service account in the argocd instance namespace
<pre>
<code>
for namespace in ccop-ref-dev,ccop-ref-uat,ccop-ref-prod
do
    oc adm policy -n $namespace add-cluster-role-to-user cluster-admin system:serviceaccount:<argocd instance namespace>:<instance name>-argocd-application-controller
done;
</code>
</pre>
or
<pre>
<code>
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cluster-admin-argocd-argocd-application-controller
subjects:
  - kind: ServiceAccount
    name: argocd-argocd-application-controller
    namespace: <argocd instance namespace>
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
</code>
</pre>
</li>

<li>Create the applications to deploy. Kindly read further to see how you can achieve this</li>

</ul>
<br/>
<h3>Deployment on Dev Environment</h3>
<br/>
<p>Kindly carry out the following activities:-</p>

<ul>
<li>Kindly setup the namespace <code>ccop-ref-dev</code> (if it does not exist) on Openshift for deployment</li>
<li>Log into  Argo CD cluster, go to <code>Settings/Repositories</code> and click on the button <code>+ CONNECT REPO USING HTTPS</code> in order to permit connection to your k8s repo. Kindly see screenshots below:-
<img src="argocd-repo-create.png">
<img src="argocd-repo-list.png">
</li>
<li>Kindly create the following on the dev environment:-
  <pre>
    <code>
    apiVersion: argoproj.io/v1alpha1
    kind: AppProject
    metadata:
        name: dev-project
        namespace: <argocd instance namespace>
    spec:
        clusterResourceWhitelist:
            - group: '*'
            kind: '*'
        destinations:
            - namespace: ccop-ref-prod
            server: 'https://kubernetes.default.svc'
        sourceRepos:
            - 'https://gitlab.xxx.corp.xxx.ca/AI/aiocp/tekton-pipeline.git'
        status: {}
    </code>
    </pre>

  <pre>
    <code>
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
        name: quarkus-app-dev
    spec:
        destination:
            namespace: ccop-ref-dev
            server: 'https://kubernetes.default.svc'
        source:
            path: k8s/overlays/dev
            repoURL: 'https://gitlab.bell.corp.xxx.ca/AI/aiocp/tekton-pipeline.git'
            targetRevision: dev
        project: dev-project
        syncPolicy:
            automated:
            prune: true
            selfHeal: true
    </code>
    </pre>
</li>
</ul>

<br/>
<h3>Deployment on UAT Environment</h3>
<br/>
<p>Kindly carry out the following activities:-</p>

<ul>
<li>Kindly setup the namespace <code>ccop-ref-uat</code> (if it does not exist) on Openshift for deployment</li>
<li>Kindly create the following on the dev environment:-
  <pre>
    <code>
    apiVersion: argoproj.io/v1alpha1
    kind: AppProject
    metadata:
        name: uat-project
        namespace: <argocd instance namespace>
    spec:
        clusterResourceWhitelist:
            - group: '*'
            kind: '*'
        destinations:
            - namespace: ccop-ref-uat
            server: 'https://kubernetes.default.svc'
        sourceRepos:
            - 'https://gitlab.xxx.corp.xxx.ca/AI/aiocp/tekton-pipeline.git'
        status: {}
    </code>
    </pre>

  <pre>
    <code>
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
        name: quarkus-app-dev
    spec:
        destination:
            namespace: ccop-ref-dev
            server: 'https://kubernetes.default.svc'
        source:
            path: k8s/overlays/uat
            repoURL: 'https://gitlab.xxx.corp.xxx.ca/AI/aiocp/tekton-pipeline.git'
            targetRevision: uat
        project: uat-project
        syncPolicy:
            automated:
            prune: true
            selfHeal: true
    </code>
    </pre>
</li>
</ul>


<br/>
<h3>Deployment on Prod Environment</h3>
<br/>
<p>Kindly carry out the following activities:-</p>

<ul>
<li>Kindly setup the namespace <code>ccop-ref-prod</code> (if it does not exist) on Openshift for deployment</li>
<li>Kindly create the following on the dev environment:-
  <pre>
    <code>
    apiVersion: argoproj.io/v1alpha1
    kind: AppProject
    metadata:
        name: prod-project
        namespace: <argocd instance namespace>
    spec:
        clusterResourceWhitelist:
            - group: '*'
            kind: '*'
        destinations:
            - namespace: ccop-ref-prod
            server: 'https://kubernetes.default.svc'
        sourceRepos:
            - 'https://gitlab.xxx.corp.xxx.ca/AI/aiocp/tekton-pipeline.git'
        status: {}
    </code>
    </pre>

  <pre>
    <code>
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
        name: quarkus-app-prod
    spec:
        destination:
            namespace: ccop-ref-dev
            server: 'https://kubernetes.default.svc'
        source:
            path: k8s/overlays/prod
            repoURL: 'https://gitlab.xxx.corp.xxx.ca/AI/aiocp/tekton-pipeline.git'
            targetRevision: master
        project: prod-project
        syncPolicy:
            automated:
            prune: true
            selfHeal: true
    </code>
    </pre>
</li>
</ul>
