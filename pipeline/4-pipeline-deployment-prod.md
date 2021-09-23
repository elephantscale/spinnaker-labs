# Production Deployments


Here we will going going over Deployments

### Background: ReplicaSets vs. Deployments, and Services

At this point, it may be helpful to get a little bit more background about Kubernetes ReplicaSets, Deployments, and Services in Kubernetes

#### ReplicaSets

In Kubernetes, a `ReplicaSet` object is an object that defines an identical set of pods.

Here's how this works:

* The section under `spec.template` is the pod spec - this defines how Kubernetes should create each of the individual pods
* The pod spec has some set of labels, that get attached to each of the pods owned by the ReplicaSet
* The ReplicaSet also has two other relevant fields:
  * The `spec.replicas` field indicates how many _replicas_, or instances, of the pod to create
  * The `spec.selector.matchLabels` dictionary defines how the ReplicaSet identifies the pods that are owned by it.
* In the Kubernetes cluster, there is a ReplicaSet controller that looks at all of the Kubernetes ReplicaSets and ensures that Pod objects get created in the Kubernetes cluster to match.
* Each Pod that is owned by a ReplicaSet must have all of the labels indicated in the ReplicaSet's `spec.selector.matchLabels` dictionary.  The pods may have additional fields that are not in the `matchLabels` dictionary.
* Therefore, if you want some set of pods to match some pod definition, you can create a ReplicaSet and the ReplicaSet controller will create those objects for you.

The key point here is that a ReplicaSet defines a set of identical pods, which have the same pod spec.  A ReplicaSet's pod spec **cannot** be changed (aside from adding/removing labels and metadata - more on this later) - you cannot, for example, change the Docker image used for a given ReplicaSet.  

It may be helpful to think of a ReplicaSet managing or _owning_ a set of Pods

#### Deployments

Because of the above limitations on changing ReplicaSets, the Kubernetes Deployment object was created.

In Kubernetes, a `Deployment` object is an object that is used to orchestrate ReplicaSets.

Here's how this works:

* A Deployment object will look very similar to a ReplicaSet, with a few key differences
* A Deployment's pod spec **can** be changed
* The Deployment object will be responsible for the creation and manipulation of ReplicaSets
  * There is a Kubernetes Deployment controller that looks at your Deployment objects and creates ReplicaSets to match
  * For example, when you first define a Deployment, Kubernetes will take the pod spec and use it to create a ReplicaSet that has the same pod spec
  * If you then update that Deployment with a different Pod spec, Kubernetes will create a new ReplicaSet matching the new Pod spec
  * Kubernetes will also handle scaling up the new ReplicaSet and scaling down the old ReplicaSet, according to other metadata in the Deployment object (rollout strategies)
* Each pod that gets created by the ReplicaSets that are owned by a given Deployment will (and must) a set of labels including all of the labels in the Deployment object's  `spec.selector.matchLabels` dictionary.  They may have additional labels.

It's helpful to think of a Deployment managing or _owning_ a set of ReplicaSets

#### Services

A Kubernetes `Service` is essentially an internal load balancer (or reverse proxy) in Kubernetes.  By default, a Kubernetes Service will create an internal IP address (accessibile within the Kubernetes cluster) through which a set of pods are load balanced (there are also options to create external load balancers).

Here's how this works:

* A Service has a `spec.selector` field that has a set of Kubernetes labels.
* The Service will load balance all pods that have all of the labels on the spec.selector.

#### Example 1: Deployment (Rollng Deployment)

Say we have the following resources:

* A Deployment "backend"
  * Which creates pods with the following labels:
    * `application: backend`
    * `lb: my-backend`
  * a which has the following matchLabels:
    * `application: backend`

* A Service "my-backend-loadbalancer" with the following selector:
  * `lb: my-backend`

The service will load balancer all of the pods, because they all have the labels specified in the selector.

Then, if we want to update to a new version of our application, we can update the pod spec with the new Docker image, and Kubernetes will stand up a new ReplicaSet with the new pod spec, and scaled down the old ReplicaSet to 0 instances.  This is a rolling deployment, and this is what we did for the dev and test deployments in our pipeline.

#### Example 2: ReplicaSet (Blue / Green Deployment)

Say we have the following resources:

* A ReplicaSet "frontend-001"
  * Which creates pods with the following labels:
    * `application: frontend`
    * `version: 001`
    * `lb: my-loadbalancer`
    * `status: active`
  * a which has the following matchLabels:
    * `application: frontend`
    * `version: 001`

* A second ReplicaSet "frontend-002"
  * Which creates pods with the following labels:
    * `application: frontend`
    * `version: 002`
    * `lb: my-loadbalancer`
  * and which has the following matchLabels:
    * `application: frontend`
    * `version: 002`

* A Service "my-frontend-loadbalancer" with the following selector:
  * `lb: my-loadbalancer`

* A second Service "my-frontend-active-loadbalancer" with the following selector:
  * `status: active`

Then the we will have the following behavior:
* The service "my-frontend-loadbalancer" will load balance all pods from both ReplicaSets, because all pods have the labels in its selector
* The service "my-frontend-aftive-loadbalancer" will load balance only pods from the ReplicaSet "frontend-001", because only those pods match the selector `status: active`
* The ReplicaSets are able to track which pods they own, **completely independent of load balancing behavior.**

In order to perform a blue/green deployment with "my-frontend-active-loadbalancer", we have two options:

* We can modify the selector on the loadBalancer (for example, change it from `status: active` to something else)
* We can modify the labels on the ReplicaSet (for example, add the `status:active` label to the "frontend-002" ReplicaSet and remove the label from the "frontend-001" ReplicaSet)

Spinnaker has a native capability to perform this latter option.  This is what we did for our production pipeline rollout.

### Spinnaker: ReplicaSets vs. Deployments

In Kubernetes, Deployments can be used to handle the rollouts of your application: if you have version A of your application, and want to update it to version B of your application, you update your Deployment specification and Kubernetes will handle creating / updating the ReplicaSets to handle the upgrade in a rolling fashion.

Spinnaker is able to directly use this mechanism, and when you use Spinnaker to create Deployments, your pods will follow whatever pattern Kuberntes uses to meet your Kubernetes Deployment pattern.  Spinnaker also adds annotations to be able to roll back to previous iterations (we'll see this later on).

If we want more control of our application rollout - for example to do a Blue/Green deployment - then Spinnaker can also directly interact with Kubernetes ReplicaSets.  Then, if we add the proper annotations to our ReplicaSets to indicate what Service they *should* be dynamically attached to / detached from, then Spinnaker will do the following:

* Look at the desired Service
* Look at the selector on the desired Service
* If attaching the ReplicaSet pods to the Service, add all of the relevant labels to the ReplicaSet pod spec
* If detaching the ReplicaSet from the Service, remove all of the relevant labels from the ReplicaSet pod spec.

---

## Adding parameters to indicate the number of instances for production

Now, we're going to add an additional parameter to our pipeline, and use those parameters to define how many instances of our app we want in production.

1. Log into the Spinnaker UI
1. Go to the "Applications" tab
1. Click on our "hello-today" application
1. Go to the "Pipelines" tab.

Add the parameter to your pipeline:

1. Click on the "Configure" button next to your pipeline (or click on "Configure" in the top right, and select your pipeline)
1. Click on the "Configuration" icon on the left side of the pipeline
1. Add a 'prod_count' parameter:
    1. Click "Add Parameter" (in the middle of the page)
    1. Specify "prod_count" as the Name
    1. Check the "Required" checkbox
    1. Check the "Pin Parameter" checkbox
    1. Add a Default Value of "4"
1. Click "Save Changes"

Update the prod manifest to use your parameter:

1. Click on the "Deploy Prod" stage
1. In the large Kubernetes manifest field, find the `replicas` field
1. Replace the `replicas` field with this:

    ```yml
    replicas: '${ #toInt( parameters.prod_count ) }'
    ```

1. Click "Save Changes"

Then, trigger the pipeline:

1. Click back on the "Pipelines" tab at the top of the page
1. Click on "Start Manual Execution" next to your newly created pipeline (you can also click "Start Manual Execution" in the top right, and then select your pipeline in the dropdown)
1. Specify an image tag (`monday`, `tuesday`, `wednesday`, `thursday`, `friday`, `saturday`, or `sunday`)
1. Specify some number of instances for your prod deployment
1. Click "Run"

When your pipeline gets to the prod deployment (you'll have to approve the manual judgment), notice how the number of instances in the prod deployment match your parameter.  Note the syntax here - we're providing this field as a string in the parameter, and have to explicitly convert it to an integer in the SpEL (Spring Expression Language) field.  In this case, we are using a helper function provided by Spinnaker - more of these are documented on the Spinnaker document [Pipeline Expression Reference](https://www.spinnaker.io/reference/pipeline/expressions/).

## Adding an option to skip the test environment, using a parameter

We can use logic to determine whether a stage gets run or not.

1. Log into the Spinnaker UI
1. Go to the "Applications" tab
1. Click on our "hello-today" application
1. Go to the "Pipelines" tab.

Add the parameter to your pipeline:

1. Click on the "Configure" button next to your pipeline (or click on "Configure" in the top right, and select your pipeline)
1. Click on the "Configuration" icon on the left side of the pipeline
1. Add a 'prod_count' parameter:
    1. Click "Add Parameter" (in the middle of the page)
    1. Specify "deploy_test" as the Name
    1. Select the "Required" checkbox
    1. In the "Default Value" field specify `yes`
    1. Select the "Show Options" checkbox
    1. In the "Options" section, enter these two values (press "Add New Option" to add the second one):
      * `yes`
      * `no`
1. Click "Save Changes"

Then, configure the stage to be skippable:

1. Click on the "Deploy Test" stage
1. Scroll down to the bottom, and check the "Conditional on Expression" checkbox
1. In the field that pops up, specify this:

    ```java
    ${parameters['deploy_test'] == 'yes'}
    ```

1. Click "Save Changes"

Then, trigger the pipeline:

1. Click back on the "Pipelines" tab at the top of the page
1. Click on "Start Manual Execution" next to your newly created pipeline (you can also click "Start Manual Execution" in the top right, and then select your pipeline in the dropdown)
1. Specify an image tag (`monday`, `tuesday`, `wednesday`, `thursday`, `friday`, `saturday`, or `sunday`)
1. Specify some number of instances for your prod deployment
1. Select "no" in the 'deploy_test' field
1. Click "Run"

Note these things:

* You can, generally, reference parameters (and other fields using either dot notation (`parameters.deploy_test`) or dictionary lookup notation (`parameters['deploy_test']`).
* The help text for the conditional on expression stage may say you don't need the `${}`.  You should still use it.
* Spring Expression Language expects Java syntax between the curly brackets: `${JAVA EXPRESSION HERE}`
* When you trigger the pipeline through the UI, you will get the dropdown options.  Later on, when we trigger the pipeline via a webhook trigger, you won't get these options - if you don't specify *exactly* `yes`, the stage will get skipped.  An alternative option for the conditional to only skip the stage on a 'no' might be:
  * `${parameters['deploy_test'] != 'no'}`
  * More complex conditionals, such as those that are case insensitive

