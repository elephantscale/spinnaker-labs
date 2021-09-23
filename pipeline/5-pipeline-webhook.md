# Webhook Invocation

Here we will make a webhook trigger for our application


## Adding a webhook trigger to our application

Now, let's configure our pipeline so that it can be triggered from an external system.  There are a number of ways to achieve this; in this document, we will cover two different types of webhooks.

The first mechanism is to expose the pipeline on an **unauthenticated** webhook endpoint (note that we can still specify _plaintext_ verification fields that can be used to prevent the pipeline from being triggered by just anybody).

1. Log into the Spinnaker UI
1. Go to the "Applications" tab
1. Click on our "hello-today" application
1. Go to the "Pipelines" tab.

Add the parameter to your pipeline:

1. Click on the "Configure" button next to your pipeline (or click on "Configure" in the top right, and select your pipeline)
1. Click on the "Configuration" icon on the left side of the pipeline
1. Scroll down to the "Automated Triggers" section of the pipeline
1. Click "Add Trigger"
1. In the dropdown, select "Webhook trigger"
1. In the "Source" field, type `my-first-pipeline`.  Note that under the field, a URL will get populated that will look like this:
`https://<your-minnaker-url>/api/v1/webhooks/webhook/my-first-pipeline`.  Record this URL.
1. Click "Save Changes"

Now, trigger the pipeline:

1. Go to a bash/shell terminal (this can be run from Minnaker itself), and run this command:

```bash
curl -k -X POST -H 'content-type:application/json' -d '{}' https://<your-minnaker-url>/api/v1/webhooks/my-first-pipeline
```

1. This is the same command, spread out across multiple lines (`\` allows you to split a shell command into multiple lines, as long as there are no spaces after the `\`) - you can run this instead.

```bash
    curl -k \
      -X POST \
      -H 'content-type:application/json' \
      -d '{}' \
      https://<your-minnaker-url>/api/v1/webhooks/my-first-pipeline
```

    These are what each of the flags mean:

    * `-k`: Skip TLS certificate validation (since Minnaker uses a self-signed certificate)
    * `-X POST`: Submit an HTTP POST instead of the default GET
    * `-H 'content-type:application/json'`: Add an HTTP header with key `content-type` and value `application/json` indicating that we're submitting a JSON object to the API
    * `-d '{}'`: The actual JSON object that we're submitting (in this case, an empty JSON object).  Note the single quotes surrounding the JSON object; this is so we can use double quotes inside the object without escaping.
    * The actual URL to submit the request to.

1. In the Spinnaker UI, note that the pipeline gets started.  Also note that the "Deploy Test" stage runs - this is because the default value for that parameter is "yes" (unless you've changed it).

1. You can also trigger the pipeline with parameters:

```bash
    curl -k \
      -X POST \
      -H 'content-type:application/json' \
      -d '{
            "parameters": {
              "tag":"tuesday",
              "prod_count": "5"
            }
          }' \
      https://<your-minnaker-url>/api/v1/webhooks/my-first-pipeline
```

Now, add a payload constraint to the trigger:

1. Go back to your pipeline configuration, to the trigger section
1. Click "Add payload constraint" under your webhook trigger.
1. Specify the "key" as `secret` and the value as `hunter2`

And trigger our pipeline (again, from a bash/shell terminal)

```bash
  curl -k \
    -X POST \
    -H 'content-type:application/json' \
    -d '{
          "secret": "hunter2",
          "parameters": {
            "tag":"tuesday",
            "prod_count": "5"
          }
        }' \
    https://<your-minnaker-url>/api/v1/webhooks/my-first-pipeline
```

Also try the following, which will return a successful response, but will fail to trigger the pipeline because we don't have the validator field:

```bash
  curl -k \
    -X POST \
    -H 'content-type:application/json' \
    -d '{
          "parameters": {
            "tag":"tuesday",
            "prod_count": "5"
          }
        }' \
    https://<your-minnaker-url>/api/v1/webhooks/my-first-pipeline
```

This is because a given webhook will only trigger a given pipeline if the webhook payload matches all of the payload constraints on the trigger.

Be aware of these things when setting up webhook triggers:

* Webhook trigger endpoints are essentially unauthenticated.  You can add validator fields, but these are plaintext in the Spinnaker pipeline configuration and should not be considered a security feature.
* You can have multiple pipelines triggering off the same endpoint.  You can, optionally, differentiate these by different payload constraints on different pipelines (for example, you could have a payload constraint that is `environment`, and configure different pipelines to trigger based on what the value of that payload constraint is).
* You can have multiple webhook triggers for a given pipeline.
* You can inject additional metadata into the pipeline that isn't exposed through a parameter, through additional fields in the trigger.  All fields in the triggering webhook will be available for usage in the pipeline through SpEL.

## Triggering a pipeline without a wehook trigger, with a webhook

In addition to the above trigger, every time you click "Start Manual Execution" through the Spinnaker UI, your browser is making an API call, acting as you, to the Spinnaker (Gate) API.  You can use the same API to trigger pipelines in an authenticated manner.

**Limitation: This only works if you are able to authenticate, which is trivial when your Spinnaker cluster is configured with basic auth or LDAP, and much harder when it's configured with SAML or OAuth2.0 authentication.  If using one of the latter two, you must set up x509 client certificate authentication to use this capability**

Here's how you do this:

1. Come up with your basic auth string.  Take your **Basic Auth** or **LDAP** username/password combination, with a colon between them, and run them through base64.  For example:

```bash
$ echo -n 'admin:my-new-password' | base64
YWRtaW46bXktbmV3LXBhc3N3b3Jk
```

_(The `-n` flag is key, as otherwise echo adds an endline to the output, which creates an invalid auth token)_

```bash
curl -k \
  -X POST \
  -H 'authorization: Basic YWRtaW4vbXktbmV3LXBhc3N3b3Jk' \
  -H 'content-type: application/json' \
  -d '{"parameters": {"tag": "tuesday"}}' \
  'https://<your-minnaker-url>/api/v1/pipelines/v2/hello-today/Deploy%20Application'
```

Be aware of these things when using this:

* The path for this is `https://<your-minnaker-url>/api/v1/pipelines/v2/<application-name>/<urlencoded-pipeline-name>`
* For pipelines that have special characters, you should use the URL encoded version of those characters (for example, `' '` (space) is `%20`)
* You cannot pass in arbitrary metadata that doesn't come in in either the `parameters` or `artifacts` fields.
* You must be using LDAP or Basic Auth
* If your cluster is configured for SAML or OAuth2.0, you must also set up X509 client certs, and use that instead

