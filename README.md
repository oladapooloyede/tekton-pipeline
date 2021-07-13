<h1>Continuous Integration and Continuous Delivery (CICD) on OpenShift - Reference Implementation </h1>

<br/>

This article will attempt to walk you through a Reference Implementation of OpenShift Continuous Integration and Continuous Delivery (CICD) using a sample Quarkus project on the Dev environment(We attempt to create an article for other environments like UAT and Prod in the future but an understanding this article might make the future articles not so necessary <span style="font-size:12px">&#128521;</span> ). For the sake of simpicity, the article will be broken down into 2 fascicles. They are as follows:-

<Ul>
    <li><a href="#ci">Continuous Integration Reference Implementation on OpenShift using OpenShift Pipelines (Tekton Project)</a></li>
    <li>Continuous Delivery Reference Implementation on OpenShift using OpenShift GitOps </li>
</Ul>

<br/>

<h2 id="ci">Continuous Integration Reference Implementation on OpenShift using OpenShift Pipelines (Tekton Project)</h2>

<br/>

OpenShift Pipelines are based on the Tekton project (https://tekton.dev) - a new, Kubernetes and container native way to manage pipelines.

The purpose of this article is demonstrate a reference implementation for the Openshift Pipelines using a sample Quarkus project.

If everything works well, the pipeline will look like this:-
<br/>

<img src="pipeline.png"></img>

<br/>

The pipeline will:
<Ul>
    <li>Clone the source code repository (https://github.com/oladapooloyede/Second-quarkus-reactive-project.git) on commit to <code>dev</code> branch</li>
    <li>Run test cases</li>
    <li>Build artifact for image</li>
    <li>In parallel scan build artifact for vulnerabilities using a <a href="https://github.com/aquasecurity/trivy"><code>Trivy</code></a> task </li>
    <li>Build an tag image using the source code <code>commitId</code></li>
    <li>In parallel scan image locally for vulnerabilities using a <a href="https://github.com/aquasecurity/trivy"><code>Trivy</code></a> task</li>
    <li>Push image to image repository - artifactory.xxxx.corp.xxx.ca:5073 (<code>ccop-dev</code> repo)</li>
    <li>Scan image in remote image repository for vulnerabilities using a <a href="https://github.com/aquasecurity/trivy"><code>Trivy</code></a> task</li>
    <li>Update the repository https://github.com/oladapooloyede/tekton-pipeline.git under the path <code>k8s/overlays/dev</code> to point to the latest image in the Artifactory registry</li> 
</Ul>

<br/>
The Goal of this of article is:
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
    <li>Create or get access to the reference source code repository (https://github.com/oladapooloyede/Second-quarkus-reactive-project.git)</li>
    <li>Create or get access to the reference k8s repository for continuous deployment (https://github.com/oladapooloyede/tekton-pipeline.git)</li>
    <li>From your profile page, in the source code repository kindly setup an Access Token with the right permissions as shown in the screenshot below:-<img src="pat.png"></img>
    (Kindly note that the same Access Token was used for the k8s repository in the reference implementation. If you have different user profiles, you will have to setup multiple Access tokens)</li>
    <li>Setup a secret for the Access Token(s) in the Openshift namespace (<code>cop-pipeline</code>) and annotate the secret(s) (<code>tekton.dev/git-0: 'https://gitlab.xxx.corp.xxx.ca</code>) as shown in the yaml file below:-
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
    selfLink: /api/v1/namespaces/dapo-reference-pipeline/serviceaccounts/pipeline
    uid: 4ac57ed2-7a68-4b4e-bf61-4c44b44f4d01
    resourceVersion: '133440'
    creationTimestamp: '2021-07-02T19:16:06Z'
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

