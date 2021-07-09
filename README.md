<h1>Continuous Integration and Continuous Delivery (CICD) on OpenShift - Reference Implementation</h1>

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

If everything works well, the 