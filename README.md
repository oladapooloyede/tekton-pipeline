# Continuous Integration and Continuous Delivery (CICD) on OpenShift - Reference Implementation

<br/>

This article will attempt to walk you through a Reference Implementation of OpenShift Continuous Integration and Continuous Delivery (CICD) using a sample Quarkus project as a refe. For the sake of simpicity, the article will be broken down into 2 fascicles. They are as follows:-

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


### Prerequisites
<br/>

The following are required to run this reference pipeline (and possibly your pipeline)

<Ul>
    <li>Create an Openshift namespace - (In this case - <code>cop-pipeline</code>)</li>
    <li>Create or get access to the reference source code repository (https://github.com/oladapooloyede/Second-quarkus-reactive-project.git)</li>
    <li>Create or get access to the reference k8s repository for deployment (https://github.com/oladapooloyede/tekton-pipeline.git)</li>
</Ul>

    In the source code repository kindly create the following 
    In the OpenShift namespace ( i.e. <code>cop-pipeline</code>), kindly