- create a `run.sh` at the root
- make it executable `chmod +x run.sh`
- put the following in it, replacing with real values:
  ```sh
  docker run -d --restart always --name github-runner \
  -e REPO_URL="https://github.com/myoung34/repo" \
  -e RUNNER_NAME="foo-runner" \
  -e RUNNER_TOKEN="footoken" \
  -e RUNNER_WORKDIR="/tmp/github-runner-your-repo" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /tmp/github-runner-your-repo:/tmp/github-runner-your-repo \
  myoung34/github-runner:latest
  ```
  - `run.sh` is ignored by git so secrets in there won't get saved in version control (just don't remove it from .gitignore!)
- now start your container with `./run.sh`

----- ORIGINAL README BELOW -----

# Docker Github Actions Runner

[![Docker Pulls](https://img.shields.io/docker/pulls/myoung34/github-runner.svg)](https://hub.docker.com/r/myoung34/github-runner)

This will run the [new self-hosted github actions runners](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/hosting-your-own-runners).

## Docker Artifacts

| Container Base | Supported Architectures  | Tag Regex                        | Docker Tags                                                                                     | Description                                                                                                                                                                                  |
| -------------- | ------------------------ | -------------------------------- | ----------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ubuntu eoan    | `x86_64`,`armv7`,`arm64` | `/\d\.\d{3}\.\d+/`               | [latest](https://hub.docker.com/r/myoung34/github-runner/tags?page=1&name=latest)               | This is the latest build (Rebuilt nightly and on master merges). Tags without an OS name are included.                                                                                       |
| ubuntu bionic  | `x86_64`,`armv7`,`arm64` | `/\d\.\d{3}\.\d+-ubuntu-bionic/` | [ubuntu-bionic](https://hub.docker.com/r/myoung34/github-runner/tags?page=1&name=ubuntu-bionic) | This is the latest build from bionic (Rebuilt nightly and on master merges). Tags with `-ubuntu-bionic` are included and created on [upstream tags](https://github.com/actions/runner/tags). |
| ubuntu xenial  | `x86_64`,`armv7`,`arm64` | `/\d\.\d{3}\.\d+-ubuntu-xenial/` | [ubuntu-xenial](https://hub.docker.com/r/myoung34/github-runner/tags?page=1&name=ubuntu-xenial) | This is the latest build from xenial (Rebuilt nightly and on master merges). Tags with `-ubuntu-xenial` are included and created on [upstream tags](https://github.com/actions/runner/tags). |

These containers are built via Github actions that [copy the dockerfile](https://github.com/myoung34/docker-github-actions-runner/blob/master/.github/workflows/deploy.yml#L47), changing the `FROM` and building to provide simplicity.

## Environment Variables

| Environment Variable | Description                                                                                                                                                                                                                                          |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `RUNNER_NAME`        | The name of the runner to use. Supercedes (overrides) `RUNNER_NAME_PREFIX`                                                                                                                                                                           |
| `RUNNER_NAME_PREFIX` | A prefix for a randomly generated name (followed by a random 13 digit string). You must not also provide `RUNNER_NAME`. Defaults to `github-runner`                                                                                                  |
| `ACCESS_TOKEN`       | A [github PAT](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) to use to generate `RUNNER_TOKEN` dynamically at container start. Not using this requires a valid `RUNNER_TOKEN`                         |
| `ORG_RUNNER`         | Only valid if using `ACCESS_TOKEN`. This will set the runner to an org runner. Default is 'false'. Valid values are 'true' or 'false'. If this is set to true you must also set `ORG_NAME` and makes `REPO_URL` unneccesary                          |
| `ORG_NAME`           | The organization name for the runner to register under. Requires `ORG_RUNNER` to be 'true'. No default value.                                                                                                                                        |
| `LABELS`             | A comma separated string to indicate the labels. Default is 'default'                                                                                                                                                                                |
| `REPO_URL`           | If using a non-organization runner this is the full repository url to register under such as 'https://github.com/myoung34/repo'                                                                                                                      |
| `RUNNER_TOKEN`       | If not using a PAT for `ACCESS_TOKEN` this will be the runner token provided by the Add Runner UI (a manual process). Note: This token is short lived and will change frequently. `ACCESS_TOKEN` is likely preferred.                                |
| `RUNNER_WORKDIR`     | The working directory for the runner. Runners on the same host should not share this directory. Default is '/\_work'. This must match the source path for the bind-mounted volume at RUNNER_WORKDIR, in order for container actions to access files. |

## Examples

### Note

If you're using a RHEL based OS with SELinux, add `--security-opt=label=disable` to prevent [permission denied](https://github.com/myoung34/docker-github-actions-runner/issues/9)

### Manual

```
# org runner
docker run -d --restart always --name github-runner \
  -e RUNNER_NAME_PREFIX="myrunner" \
  -e ACCESS_TOKEN="footoken" \
  -e RUNNER_WORKDIR="/tmp/github-runner-your-repo" \
  -e ORG_RUNNER="true" \
  -e ORG_NAME="octokode" \
  -e LABELS="my-label,other-label" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /tmp/github-runner-your-repo:/tmp/github-runner-your-repo \
  myoung34/github-runner:latest
# per repo
docker run -d --restart always --name github-runner \
  -e REPO_URL="https://github.com/myoung34/repo" \
  -e RUNNER_NAME="foo-runner" \
  -e RUNNER_TOKEN="footoken" \
  -e RUNNER_WORKDIR="/tmp/github-runner-your-repo" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /tmp/github-runner-your-repo:/tmp/github-runner-your-repo \
  myoung34/github-runner:latest
```

Or shell wrapper:

```
function github-runner {
    name=github-runner-${1//\//-}
    org=$(dirname $1)
    repo=$(basename $1)
    tag=${3:-latest}
    docker rm -f $name
    docker run -d --restart=always \
        -e REPO_URL="https://github.com/${org}/${repo}" \
        -e RUNNER_TOKEN="$2" \
        -e RUNNER_NAME="linux-${repo}" \
        -e RUNNER_WORKDIR="/tmp/github-runner-${repo}" \
        -e LABELS="my-label,other-label" \
        -v /var/run/docker.sock:/var/run/docker.sock \
        -v /tmp/github-runner-${repo}:/tmp/github-runner-${repo} \
        --name $name ${org}/github-runner:${tag}
}

github-runner your-account/your-repo       AARGHTHISISYOURGHACTIONSTOKEN
github-runner your-account/some-other-repo ARGHANOTHERGITHUBACTIONSTOKEN ubuntu-xenial
```

Or `docker-compose.yml`:

```yml
version: "2.3"

services:
  worker:
    build: .
    image: myoung34/github-runner:latest
    environment:
      REPO_URL: https://github.com/example/repo
      RUNNER_NAME: example-name
      RUNNER_TOKEN: someGithubTokenHere
      RUNNER_WORKDIR: /tmp/runner/work
      ORG_RUNNER: "false"
      LABELS: linux,x64,gpu
    security_opt:
      # needed on SELinux systems to allow docker container to manage other docker containers
      - label:disable
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "/tmp/runner:/tmp/runner"
      # note: a quirk of docker-in-docker is that this path
      # needs to be the same path on host and inside the container,
      # docker mgmt cmds run outside of docker but expect the paths from within
```

### Nomad

```
job "github_runner" {
  datacenters = ["home"]
  type = "system"

  task "runner" {
    driver = "docker"

    env {
      ACCESS_TOKEN       = "footoken"
      RUNNER_NAME_PREFIX = "myrunner" \
      RUNNER_WORKDIR     = "/tmp/github-runner-your-repo"
      ORG_RUNNER         = "true"
      ORG_NAME           = "octokode"
      LABELS             = "my-label,other-label"
    }

    config {
      privileged = true
      image = "myoung34/github-runner:latest"
      volumes = [
        "/var/run/docker.sock:/var/run/docker.sock",
        "/tmp/github-runner-your-repo:/tmp/github-runner-your-repo",
      ]
    }
  }
}
```

### Kubernetes

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: actions-runner
  namespace: runners
spec:
  replicas: 1
  selector:
    matchLabels:
      app: actions-runner
  template:
    metadata:
      labels:
        app: actions-runner
    spec:
      volumes:
      - name: dockersock
        hostPath:
          path: /var/run/docker.sock
      - name: workdir
        hostPath:
          path: /tmp/github-runner-your-repo
      containers:
      - name: runner
        image: myoung34/github-runner:latest
        env:
        - name: ORG_RUNNER
          value: true
        - name: ORG_NAME
          value: octokode
        - name: LABELS
          value: my-label,other-label
        - name: RUNNER_TOKEN
          value: footoken
        - name: REPO_URL
          value: https://github.com/your-account/your-repo
        - name: RUNNER_NAME_PREFIX
          value: foo
        - name: RUNNER_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: RUNNER_WORKDIR
          value: /tmp/github-runner-your-repo
        volumeMounts:
        - name: dockersock
          mountPath: /var/run/docker.sock
        - name: workdir
          mountPath: /tmp/github-runner-your-repo
```

## Usage From GH Actions Workflow

```
name: Package

on:
  release:
    types: [created]

jobs:
  build:
    runs-on: self-hosted
    steps:
    - uses: actions/checkout@v1
    - name: build packages
      run: make all
```

## Automatically Acquiring a Runner Token

A runner token can be automatically acquired at runtime if `ACCESS_TOKEN` (a GitHub personal access token) is a supplied. This uses the [GitHub Actions API](https://developer.github.com/v3/actions/self_hosted_runners/#create-a-registration-token). e.g.:

```
docker run -d --restart always --name github-runner \
  -e ACCESS_TOKEN="footoken" \
  -e RUNNER_NAME="foo-runner" \
  -e RUNNER_WORKDIR="/tmp/github-runner-your-repo" \
  -e ORG_RUNNER="true" \
  -e ORG_NAME="octokode" \
  -e LABELS="my-label,other-label" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /tmp/github-runner-your-repo:/tmp/github-runner-your-repo \
  myoung34/github-runner:latest
```

## Create GitHub personal access token

Creating GitHub personal access token (PAT) for using by self-hosted runner make sure the following scopes are selected:

- repo (all)
- admin:org (all) **_(mandatory for organization-wide runner)_**
- admin:public_key - read:public_key
- admin:repo_hook - read:repo_hook
- admin:org_hook
- notifications
- workflow

Also, when creating a PAT for self-hosted runner which will process events from several repositories of the particular organization, create the PAT using organization owner account. Otherwise your new PAT will not have sufficient privileges for all repositories.
