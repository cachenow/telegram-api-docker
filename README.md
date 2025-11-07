# telegram-api-docker

## Fork Notice

This repository is a maintained fork of `lukaszraczylo/tdlib-telegram-bot-api-docker`.
It focuses on keeping the build workflow and multi-architecture GHCR images up to date.
All credit goes to the original author and the upstream project.
The license remains the same as in the upstream repository.

## Purpose of the project

Produce a minimal Docker image for the Telegram Bot API server together with an easy-to-use pipeline
that builds and publishes images when upstream changes are detected and on a weekly schedule.

**Motivation:** [#0](https://medium.com/swlh/building-your-home-raspberry-pi-kubernetes-cluster-14eeeb3c521e), [#1](https://github.com/tdlib/telegram-bot-api/issues/65), [#2](https://github.com/tdlib/telegram-bot-api/issues/65)

This project does not modify any part of the [tdlib/telegram-bot-api](https://github.com/tdlib/telegram-bot-api) code.

## Issues

As I do not modify any part of the server code I am not responsible for the way it works. For that purpose you should open an issue on the [telegram bot api server](https://github.com/tdlib/telegram-bot-api/issues) issue tracker.

**TL;DR:** My responsibility ends when container and binary starts.

## Build schedule
Builds are triggered automatically once a week to produce the latest available Telegram Bot API server image.

Images are versioned in format `1.0.x` where `x` is a build number.
There's additional version tag added, for example `api-5.1` where `5.1` is the version of Telegram API supported by the image.

## How to use the image

### Supported Architectures

Images published by this repository support `linux/amd64` and `linux/arm64`.
Docker will automatically pull the correct image for your host architecture.

### Github authentication

~~You may need to authenticate with github ([see this thread](https://github.community/t/docker-pull-from-public-github-package-registry-fail-with-no-basic-auth-credentials-error/16358/87)) to pull even the publicly available images. To do so you need to create [Personal Access Token](https://github.com/settings/tokens/new) with `read:packages` scope and use it to authenticate your docker client with the Github Docker Registry.~~

Update: After move to GHCR.io there's no need authenticate and you should be able to pull images without any additional magic.

### Docker configuration

```
docker pull ghcr.io/cachenow/telegram-api-docker:latest
docker run -p 8081:8081 \
  -e TELEGRAM_API_ID=yourApiID \
  -e TELEGRAM_API_HASH=yourApiHash \
  ghcr.io/cachenow/telegram-api-docker:latest
```

*Thing to remember:* Entrypoint is set to the server binary, therefore you can still modify parameters on the go, as shown below

#### Setting the log output and verbosity
![Set the log output and verbosity](img/screen-001.png?raw=true)

#### Printing out the help
![Print out the help](img/screen-002.png?raw=true)

### Kubernetes configuration

Example deployment within kubernetes cluster

```yaml
# apiVersion: v1
# kind: PersistentVolumeClaim
# metadata:
#   name: telegram-api
# spec:
#   accessModes:
#     - ReadWriteMany
#   storageClassName: longhorn
#   resources:
#     requests:
#       storage: 5Gi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: telegram-api
  labels:
    app: telegram-api
spec:
  selector:
    matchLabels:
      app: telegram-api
  replicas: 2
  template:
    metadata:
      labels:
        app: telegram-api
    spec:
      containers:
        - name: bot-api
          image: ghcr.io/cachenow/telegram-api-docker:latest
          imagePullPolicy: Always
          args: [ "--local", "--max-webhook-connections", "1000" ]
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
            limits:
              cpu: 500m
              memory: 500Mi
          ports:
            - containerPort: 8081
              protocol: TCP
              name: api
          env:
            - name: TELEGRAM_API_ID
              value: "xxx"
            - name: TELEGRAM_API_HASH
              value: "yyy"
          # volumeMounts:
          # - name: shared-storage
          #   mountPath: /data
      # volumes:
      # - name: shared-storage
      #   persistentVolumeClaim:
      #     claimName: telegram-api
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/control-plane
                operator: DoesNotExist
              - key: node-role.kubernetes.io/storage
                operator: DoesNotExist
              - key: node-role.kubernetes.io/highmem
                operator: Exists
```
## Tags

Depending on how a build is triggered, images may be published with some or all of the following tags:

- `latest` — multi-arch manifest pointing to the most recent stable build
- `edge` — default tag for CI builds that are not tied to a release
- `nightly` — scheduled weekly build when triggered by the scheduler
- Semantic version (e.g. `9.2.0`) when a release tag is present upstream
- `api-<major.minor>` — derived from the upstream CMake project version
- Short commit SHA and run number tags for traceability

All tags are multi-architecture manifests built from native `amd64` and `arm64` runners.

## Credits

- Upstream server: [tdlib/telegram-bot-api](https://github.com/tdlib/telegram-bot-api)
- Original Docker work: [lukaszraczylo/tdlib-telegram-bot-api-docker](https://github.com/lukaszraczylo/tdlib-telegram-bot-api-docker)
