# Blue Green Deployment of Pipeline

Let us show a blue/green deployment, which is handled by Spinnaker's traffic management capabilities.  We're going to gate this with both a manual judgment and a wait stage.

Go back to the Spinnaker pipelines page:

1. Log into the Spinnaker UI
1. Go to the "Applications" tab
1. Click on our "hello-today" application
1. Go to the "Pipelines" tab.

Edit your pipeline:

1. Click on the "Configure" button next to your pipeline (or click on "Configure" in the top right, and select your pipeline)
1. We're going to add two stages that depend on the "Deploy Test" stage.
    1. Add the Manual Judgment Stage:
        1. Click on the "Deploy Test" stage
        1. Click on "Add stage"
        1. Select "Manual Judgment" from the "Type" dropdown
        1. In the "Stage Name", enter "Manual Judgment: Deploy to Prod"
        1. In the "Instructions" field, enter "Please verify Test and click 'Continue' to continue deploying to Prod"
        1. Click "Save Changes" in the bottom right.
    1. Add the Wait Stage:
        1. Click on the "Deploy Test" stage
        1. Click on "Add stage"
        1. Select "Wait" from the "Type" dropdown
        1. In the "Stage Name", enter "Wait 30 Seconds"
        1. Click "Save Changes" in the bottom right.
        1. _Notice how we now have two stages that "Depend On" the "Deploy Test" stage.  Once the "Deploy Test" stage finishes, both of these stages will start.  A stage can have one or more stages that depend on it._
1. Now we're going to add the Kubernetes blue/green "Deploy Prod" stage
    1. Click on the "Manual Judgment: Deploy to Prod" stage
    1. Click on "Add Stage"
    1. In the "Type" dropdown, select "Deploy (Manifest)"
    1. Update the "Stage Name" field to be "Deploy Prod"
    1. Click in the empty "Depends On" field, and select "Wait 30 Seconds".  _Notice how this stage depends on both the wait and manual judgment stages - it will wait till both are complete before it starts.  A stage can depend on one more or stages._
    1. In the "Account" dropdown, select "Spinnaker"
    1. Check the "Override Namespace" checkbox and select "prod" from the "Namespace" dropdown
    1. In the manifest field, enter this (_notice that this manifest is different from the other two manifests - this is explained below_).

        ```yml
        apiVersion: apps/v1
        kind: ReplicaSet
        metadata:
          name: hello-today
        spec:
          replicas: 3
          selector:
            matchLabels:
              app: hello-today
          template:
            metadata:
              labels:
                app: hello-today
            spec:
              containers:
              - image: 'justinrlee/nginx:${parameters["tag"]}'
                name: primary
                ports:
                - containerPort: 80
                  protocol: TCP
        ```

    1. Below the manifest block, go to the "Rollout Strategy Options"
    1. Check the box for "Spinnaker manages traffic based on your selected strategy"
    1. Select "prod" from the "Service Namespace" dropdown
    1. Select "hello-today" from the "Service(s)" dropdown
    1. Check the "Send client requests to new pods" checkbox
    1. Select "Red/Black" from the "Strategy" dropdown
    1. Click "Save Changes" in the bottom right.

Then, trigger the pipeline:

1. Click back on the "Pipelines" tab at the top of the page
1. Click on "Start Manual Execution" next to your newly created pipeline (you can also click "Start Manual Execution" in the top right, and then select your pipeline in the dropdown)
1. Click "Run"

