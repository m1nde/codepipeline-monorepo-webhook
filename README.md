# codepipeline-monorepo-webhook
A webhook handler that triggers AWS CodePipelines for mono repos, implemented as a serverless AWS Lambda function.

## Why
If you are building a big application consisting of multiple microservices, or you are building a data crunching application with lots of data ingestion pipelines, you probably want to use a mono repo.
Until now, this has been a pain to build properly with AWS CodePipeline, because CodePipeline still doesn't support mono repos, even though this feature has been requested [for over two years](https://forums.aws.amazon.com/thread.jspa?threadID=265045).
Let's say your application is structured like this:
```
mono-repo/
├── microservice-1/
│   ├── buildspec.yml
│   ├── handler.js
│   └── README.md
├── microservice-2/
│   ├── buildspec.yml
│   └── main.py
│ ...
```
When a commit changes files in `microservice-1`, you only want to build this specific folder, not the whole repository (this can take a long time and get expensive with a big mono repo). 
The CodePipeline WebhookFilterRule is insufficient when trying to model this workflow, because it does [not allow for regex matching](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-codepipeline-webhook-webhookfilterrule.html).
Strangely, the [Code*Build* WebhookFilter does support it](https://docs.aws.amazon.com/codebuild/latest/APIReference/API_WebhookFilter.html).

To solve this problem, create a CodePipeline for every microservice/subrepo that exists in your application and let this webhook figure out which CodePipelines need to be started.

## Deployment
You will need [Pip](https://pypi.org/project/pip/), serverless and npm. Run
```bash
pipenv install --python 3.7
npm install
```
to install the required dependencies. 

The webhook handler needs a secret string `GithubSecret` in the AWS Parameter Store to authenticate the GitHub requests.

To deploy to AWS, run
```bash
serverless deploy --env prod
```

Add the URL of the Lambda function to your mono repo on Github via _Settings - Webhook - Add webhook_. Add the value of `GithubSecet` to the webhook under _Secret_.

Check if everything works as expected by looking at the response to the initial ping request sent by GitHub. If the response says 'Ping received', the webhook handler is ready. To debug problems, first look at the response to the webhook request, it'll inform you which CodePipelines have been started and which CodePipelines could not be found. For more in-depth debugging, look at the Cloudwatch logs for the lambda function.

## Configuration
To accommodate different types of project structures we have implemented a few different configuration options:

* `branch_routing`: Within Git we usually have a branch called master and another called dev (and more depending on your project). On AWS we do not have a master and a dev environment, but we have a production and a staging environment. To support this we allow you to route branches to different pipelines based on the routes you specify.
  * `route`: If set to `'prefix'`, the name of the branch is prepended to the names of the subfolders when building the name of the CodePipelines. If set to `'postfix'`, the name of the branch is appended to the names of the subfolders when building the name of the CodePipelines. If set to `false`, the name of the branch is not prepended or appended.
  * `routing`: Allows you to specify which branches you want to redirect to what environments. Let's say you want to route the master branch to the prod and the dev branch to the staging environment, you can create this list as follows:
  ```yml
  routing:
    - master: 'prod'
    - dev: 'staging'
  ```

* `project`: In the [Structure-1](#Structure-1) of this README we have shown one type of project structure. We also support nested projects ([Structure-2](#Structure-2)) like the one here:
  ```
  mono-repo/
  ├── service-1/
  │   ├── microservice-1/
  │   │   ├── buildspec.yml
  │   │   ├── handler.js
  │   │   └── README.md
  │   └── microservice-2/
  │       ├── buildspec.yml
  │       └── main.py
  └── service-2/
      └── microservice-3/
          ├── buildspec.yml
          ├── handler.js
          └── README.md
  ```

  To configure your project structure you can use these configuration options:

  * `nested`: If set to `true`, you can use a project structure as described above. If set to `false`, you can use a project structure as described in [Structure-1](#Structure-1).
  * `prefix_parent_folder`: If set to `true` and `nested: true`, the name of the parent folder is prepended to the names of the subfolders when building the name of the CodePipelines. For example: If you have a change in microservice-1, the name `service-1` will be prepended to `microservice-1`.
  * `prefix_repo_name`: If set to `true`, the name of the repo will be prepended to the names of the subfolders when building the name of the CodePipelines. In this case that would be `mono-repo`.

## Project Structures
### Structure-1
```
mono-repo/
├── microservice-1/
│   ├── buildspec.yml
│   ├── handler.js
│   └── README.md
├── microservice-2/
│   ├── buildspec.yml
│   └── main.py
│ ...
```

### Structure-2
```
mono-repo/
├── service-1/
│   ├── microservice-1/
│   │   ├── buildspec.yml
│   │   ├── handler.js
│   │   └── README.md
│   └── microservice-2/
│       ├── buildspec.yml
│       └── main.py
└── service-2/
    └── microservice-3/
        ├── buildspec.yml
        ├── handler.js
        └── README.md
```

## Development
Pull requests are welcome. To start development, install the dev dependencies with
```bash
pipenv install --python 3.7
```
Run tests with
```bash
pipenv run pytest
```
**TODO:**
* implement a blacklist of directories that do not map to CodePipelines