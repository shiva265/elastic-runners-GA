## The Goal
We want to automatically spin-up a GitHub Self-Hosted Runners in GCP when a workflow is triggered. We also want to tear them down at the end of the workflow.

## Plan
We are going to configure a Cloud Build Webhook Trigger to run when a job is queued. The trigger will spin-up a VM that will register a new runner. The new runner will be executed using the `--ephemeral` flag ([GitHub Docs](https://docs.github.com/en/actions/hosting-your-own-runners/autoscaling-with-self-hosted-runners#using-ephemeral-runners-for-autoscaling)). The `--ephemeral` flag makes the runner available for a single job execution, after that the runner is de-registered and we can delete the instance.![image](https://user-images.githubusercontent.com/2351518/146522423-3229a657-98ab-4c3c-9b37-e89cdb063072.png)


### Before we start
Before we start we need to create the following resources:
- A Secret in Google Secret Manager named `github-token` containing the GitHub token that cloud build will use to generate a registration token
- A secret for the webhook named `webhook-token` (this can also be created automatically on the cloud console)
- A service account used by cloudbuild `runner-bootstrap` (roles: Compute Instance Admin (beta), Logs Writer, Service Account User and a role which contains get secrets access)
- A service Account used by the VM `github-runner` (roles: Compute Instance Admin (beta), Logs Writer)

### Cloudbuild

Can be triggered by code changes but also via webhooks.
Cloudbuild can also extract information from the payload sent by the caller. So, we are going to pass the following variables to th cloud build configuration.

Create a file named `build-config.yaml` and copy the contents of build-config.yaml file in home directory.

create infra and Now we can create the trigger with following command:

```bash
PROJECT_ID="THE-PROJECT-ID"
SA="projects/${PROJECT_ID}/serviceAccounts/runner-bootstrap@${PROJECT_ID}.iam.gserviceaccount.com"
gcloud alpha builds triggers create webhook \
  --name=elastic-runner-webhook \
  --secret=projects/$PROJECT_ID/secrets/webhook-secret/versions/latest \
  --substitutions=_ACTION='$(body.action)',_JOB_NAME='$(body.workflow_job.name)'',_REPO_FULLNAME='$(body.repository.full_name)',_REPO_NAME='$(body.repository.name)',_RUNNER_LABELS='$(body.workflow_job.labels)',_TIMEOUT=600 \
  --filter='_ACTION == "queued"' \
  --service-account=$SA \
  --inline-config=build-config.yaml
```


Now that our Trigger is created let's configure the webhook on GitHub.
From your GitHub repo click on -> `Settings` -> `Webhooks`

![setting-webhooks](https://user-images.githubusercontent.com/2351518/146520036-20792153-d37c-48f7-920e-ce72d437c527.png)
Now click on `Add webhook`

![add-webhook](https://user-images.githubusercontent.com/2351518/146522839-f5a8f5c5-13ca-4de4-b32e-60a46d749632.png)

In the Payload URL paste the Cloud Build Webhook Trigger copied from the Trigger configuration on (Cloudbuilder):

![cloudbuild-webhook](https://user-images.githubusercontent.com/2351518/146521538-06a3e59a-a6a9-490e-95d6-7b0b2570418e.png)

Now scroll down and click on _"Let me select individual events."_ we only wan to get triggered when a job is queued so we only click on **Workflow jobs** make sure Active is selected and click on Add webhook

![add-webhook-2](https://user-images.githubusercontent.com/2351518/146523790-b0115443-3ec8-4965-af9b-f3149ccce178.png)

! self-hosted runner registered on github console
<img width="1269" alt="Screenshot 2022-11-11 at 5 47 38 PM" src="https://user-images.githubusercontent.com/31533789/201340712-439cf8f5-1679-4e40-b629-67d1e320c872.png">

