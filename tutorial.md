# How to push tagged Docker releases to Google Artifact Registry with a GitHub Action

Here's how I configured a [GitHub Action](https://github.com/features/actions) so that a new version issued by [GitHub's release interface](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases) will build a [Dockerfile](https://docs.docker.com/engine/reference/builder/), tag it with the version number and upload it to [Google Artifact Registry](https://cloud.google.com/artifact-registry/).

Before you attempt the steps below, you need the following: 
* A GitHub repository that contains a working Dockerfile
* The Google Cloud SDK tool [gcloud](https://cloud.google.com/sdk/gcloud/) installed and authenticated

## Create a Workload Identity Federation

The first step is to create a [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation) that will allow your GitHub Action to log in to your Google Cloud account. The instructions below are cribbed from the documentation for the [google-github-actions/auth](https://github.com/google-github-actions/auth#setting-up-workload-identity-federation) Action. You should follow along in your terminal.

The first command creates a [service account](https://cloud.google.com/iam/docs/service-accounts) with Google. I will save the name I make up, as well as my Google project id, as environment variables for reuse. You should adapt the variables here, and others as we continue, to fit your project and preferred naming conventions.

```bash
export PROJECT_ID=my-project-id
export SERVICE_ACCOUNT=my-service-account

gcloud iam service-accounts create "${SERVICE_ACCOUNT}" \
  --project "${PROJECT_ID}"
```

Enable Google's IAM API for use.

```bash
gcloud services enable iamcredentials.googleapis.com \
  --project "${PROJECT_ID}"
``` 

Create a workload identity pool that will manage the GitHub Action's roles in Google Cloud's permission system.

```bash
export WORKLOAD_IDENTITY_POOL=my-pool

gcloud iam workload-identity-pools create "${WORKLOAD_IDENTITY_POOL}" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --display-name="${WORKLOAD_IDENTITY_POOL}"
```

Get the unique identifier of that pool.

```bash
gcloud iam workload-identity-pools describe "${WORKLOAD_IDENTITY_POOL}" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --format="value(name)"
```

Export the returned value to a new variable.

```bash
export WORKLOAD_IDENTITY_POOL_ID=whatever-you-got-back
```

Create a provider within the pool for GitHub to access.

```bash
export WORKLOAD_PROVIDER=my-provider

gcloud iam workload-identity-pools providers create-oidc "${WORKLOAD_PROVIDER}" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --workload-identity-pool="${WORKLOAD_IDENTITY_POOL}" \
  --display-name="${WORKLOAD_PROVIDER}" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com"
```

Allow a GitHub Action based in your repository to login to the service account via the provider.

```bash
export REPO=my-username/my-repo

gcloud iam service-accounts add-iam-policy-binding "${SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --project="${PROJECT_ID}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/${WORKLOAD_IDENTITY_POOL_ID}/attribute.repository/${REPO}"
```

Ask Google to return the identifier of that provider.

```bash
gcloud iam workload-identity-pools providers describe "${WORKLOAD_PROVIDER}" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --workload-identity-pool="${WORKLOAD_IDENTITY_POOL}" \
  --format="value(name)"
```

That will return a string that you should save for later. We'll use it in our GitHub Action.

Finally, we need to make sure that the service account we created at the start has permission to muck around with Google Artifact Registry.

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/artifactregistry.admin"
```

To verify that worked, you can ask Google print out the permissions assigned to the service account.

```
gcloud projects get-iam-policy $PROJECT_ID \
    --flatten="bindings[].members" \
    --format='table(bindings.role)' \
    --filter="bindings.members:${SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com"
```

## Create a Google Artifact Registry repository

Go to the [Google Artifact Registry](https://console.cloud.google.com/artifacts) interface within your project. Create a new repository by hitting the buttona at the top. Tell Google it will be in the Docker format and then select a region. It doesn't matter which region. Save the name you give the repo and the region's abbreviation, which will be something like `us-west1`. 

## Create a GitHub Action

Now it's time to make your GitHub Action. You should add a new YAML file in the `.github/workflows` folder. In the authentication step you'll want to fill in your provider id, your service account id and project id. In the push step you'll need to fill in your GAR repository name and region, as well as a name for your image, which you'll need to make up on your own.

```yaml
name: Release
on:
  push:

jobs:
  docker-release:
    name: Tagged Docker release to Google Artifact Registry
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags')  # <-- Notice that I'm filtering here to only run when a tagged commit is pushed

    permissions:
      contents: 'read'
      id-token: 'write'

    steps:
      - id: checkout
        name: Checkout
        uses: actions/checkout@v2

      - id: auth
        name: Authenticate with Google Cloud
        uses: google-github-actions/auth@v0
        with:
          token_format: access_token
          workload_identity_provider: <your-provider-id>
          service_account: <your-service-account>@<your-project-id>.iam.gserviceaccount.com
          access_token_lifetime: 300s

      - name: Login to Artifact Registry
        uses: docker/login-action@v1
        with:
          registry: us-west2-docker.pkg.dev
          username: oauth2accesstoken
          password: ${{ steps.auth.outputs.access_token }}

      - name: Get tag
        id: get-tag
        run: echo ::set-output name=short_ref::${GITHUB_REF#refs/*/}

      - id: docker-push-tagged
        name: Tag Docker image and push to Google Artifact Registry
        uses: docker/build-push-action@v2
        with:
          push: true
          tags: |
             <your-gar-region>-docker.pkg.dev/<your-project-id>/<your-gar-repo-name>/<your-docker-image-name>:${{ steps.get-tag.outputs.short_ref }}
             <your-gar-region>-docker.pkg.dev/<your-project-id>/<your-gar-repo-name>/<your-docker-image-name>:latest
```

## Make a release

Phew. That's it. Not you can go to the releases panel for your repo on GitHub, punch in a new version tag like `0.0.1` and hit the big green button. That should trigger a new process in your Actions tab, where the push of the tagged commit will trigger the release.

## Extra bits

In my real world implementations, I will typically have several testing steps that precede the release job. That will ensure that the code is good to go before sending out the release.

That could look something like this:

```yaml
name: Tesst and release
on:
  push:

jobs:
  test-python:
    name: Test Python code
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - id: install
        name: Install Python, pipenv and Pipfile packages
        uses: palewire/install-python-pipenv-pipfile@v2
        with:
          python-version: 3.7

      - id: run
        name: Run tests
        run: make test

  docker-release:
    name: Tagged Docker release to Google Artifact Registry
    runs-on: ubuntu-latest
    needs: [test-python]
    if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags')

    permissions:
      contents: 'read'
      id-token: 'write'

    steps:
      - id: checkout
        name: Checkout
        uses: actions/checkout@v2

      - id: auth
        name: Authenticate with Google Cloud
        uses: google-github-actions/auth@v0
        with:
          token_format: access_token
          workload_identity_provider: <your-provider-id?
          service_account: <your-service-account>@<your-project-id>.iam.gserviceaccount.com
          access_token_lifetime: 300s

      - name: Login to Artifact Registry
        uses: docker/login-action@v1
        with:
          registry: us-west2-docker.pkg.dev
          username: oauth2accesstoken
          password: ${{ steps.auth.outputs.access_token }}

      - name: Get tag
        id: get-tag
        run: echo ::set-output name=short_ref::${GITHUB_REF#refs/*/}

      - id: docker-push-tagged
        name: Tag Docker image and push to Google Artifact Registry
        uses: docker/build-push-action@v2
        with:
          push: true
          tags: |
             <your-gar-region>-docker.pkg.dev/<your-project-id>/<your-gar-repo-name>/<your-docker-image-name>:${{ steps.get-tag.outputs.short_ref }}
             <your-gar-region>-docker.pkg.dev/<your-project-id>/<your-gar-repo-name>/<your-docker-image-name>:latest
```

If you just wanted tags to trigger releases, you could likely hook the action to run on tags rather than on every push. That would mean putting something like this at the top, and removing the `if` clause I've put on the release job to filter out typical pushes.

```yaml
on:  
  push:
    tags:
      - '*'
```

But all that's up to you. Good luck. It's a real pain to get all these ducks in a row, but once you do it you'll have a streamlined release system that can be repeated quickly and smoothly well into the future.