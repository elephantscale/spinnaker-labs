# Promote Pipeline

You should have already created your pipeline in the previous lab. 

## Run the pipeline with a different parameter

1. Click back on the "Pipelines" tab at the top of the page
1. Click on "Start Manual Execution" next to your newly created pipeline (you can also click "Start Manual Execution" in the top right, and then select your pipeline in the dropdown)
1. Replace "monday" with some other day of the week (like 'tuesday' or 'wednesday')
1. Click "Run"

## Expand the pipeline: Add manual judgment and `test` deployment

Now that we have a running pipelines, let's add a promotion to a higher environment, gated by a manual approval.

Go back to the Spinnaker pipelines page:

1. Log into the Spinnaker UI
1. Go to the "Applications" tab
1. Click on our "hello-today" application
1. Go to the "Pipelines" tab.

Edit your pipeline:

1. Click on the "Configure" button next to your pipeline (or click on "Configure" in the top right, and select your pipeline)
1. Click on the "Configuration" icon on the left side of the pipeline
1. Add the "Manual Judgment: Deploy to Stage"
    1. Click "Add stage". Note how the stage is set to run at the beginning of the pipeline.
    1. Select "Manual Judgment" from the "Type" dropdown
    1. In the "Stage Name", enter "Manual Judgment: Deploy to Test"
    1. In the "Instructions" field, enter "Please verify Dev and click 'Continue' to continue deploying to Test"
    1. Click in the "Depends On" field at the top, and select your "Deploy Dev" stage.  _Notice how this rearranges the stages so that the manual judgment stage depends on (starts *after*) the dev deployment stage._
    1. Click "Save Changes" in the bottom right.
1. Add the *Deploy Test* stage
    1. In the pipeline layout section at the top of the page, click on "Manual Judgment: Deploy to Test" (you're probably already here)
    1. Click "Add stage".  _Notice how the stage is dependent on the stage you had selected when you added the stage (the manual judgment stage)._
    1. In the "Type" dropdown, select "Deploy (Manifest)"
    1. Update the "Stage Name" field to be "Deploy Test"
    1. In the "Account" dropdown, select "spinnaker"
    1. Select the 'Override Namespace' checkbox, and select 'test' in the dropdown
    1. In the "Manifest" field, put this (note the `${parameters["tag"]}` field, which will pull in the tag parameter)

        ```yml
        apiVersion: apps/v1
        kind: Deployment
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
                lb: hello-today
            spec:
              containers:
                - image: 'justinrlee/nginx:${parameters["tag"]}'
                  name: primary
                  ports:
                    - containerPort: 80
        ```

1. Click "Save Changes"

Then, trigger the pipeline:

1. Click back on the "Pipelines" tab at the top of the page
1. Click on "Start Manual Execution" next to your newly created pipeline (you can also click "Start Manual Execution" in the top right, and then select your pipeline in the dropdown)
1. Click "Run"

_Notice that we used the exact same manifest; we just selected a different override namespace.  This works because the manifest doesn't have hardcoded namespaces._

Right now, we only have one Kubernetes "Account", called "spinnaker", which refers to the Kubernetes cluster that Spinnaker is running on.

If we have added additional Kubernetes clusters to Spinnaker, we could also (alternately or in addition) configure Spinnaker to deploy to a different Kubernetes cluster by selecting a different option in the "Account" dropdown.


