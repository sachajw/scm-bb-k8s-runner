- [Bitbucket Runner Job for Kubernetes](#bitbucket-runner-job-for-kubernetes)
  - [Prerequisites](#prerequisites)
  - [Create the Bitbucket Runner](#create-the-bitbucket-runner)
  - [Monitor the Job](#monitor-the-job)
  - [Summary](#summary)

# Bitbucket Runner Job for Kubernetes

This guide provides instructions on how to use the Bitbucket Runner job in your on-prem Kubernetes cluster to execute Bitbucket Pipelines builds securely.

## Prerequisites

1. **Kubernetes Cluster**: Ensure you have access to an on-prem Kubernetes cluster.
2. **Bitbucket Runner Secrets**: You need to create a Kubernetes secret containing the following sensitive information:
   - accountUuid
   - runnerUuid
   - oauthClientId
   - oauthClientSecret
3. In Bitbucket.org go to `Workspace Settings` by clicking on the cog next to your profile icon in the top right of your screen.

![bb workspace settings](/images/01-bb-workspace-settings.png)

4. In the left hand menu scroll down until you see `Workspace runners` under the `Pipelines` heading

![bb workspace runners](/images/02-bb-workspace-runners.png)

5. Click `Add runner`, give it a name then click next and you will be given the **accountUuid, runnerUuid, oauthClientId and oauthClientSecret** to be used in the next steps.

6. Because this runner is created under the workspace settings all repos and projects can make use of the on-prem runner

## Create the Bitbucket Runner

1. Update the variables in `bb-runner-vars.txt` with your values

```shell
export ACCOUNT_UUID="your value goes here between the double quotes"
export RUNNER_UUID="your value goes here between the double quotes"
export OAUTH_CLIENT_ID="your value goes here between the double quotes"
export OAUTH_CLIENT_SECRET="your value goes here between the double quotes"

export BASE64_OAUTH_CLIENT_ID=$(echo -n $OAUTH_CLIENT_ID | base64)
export BASE64_OAUTH_CLIENT_SECRET=$(echo -n $OAUTH_CLIENT_SECRET | base64)
```

1. Copy the all the `export` values and paste them in your terminal and hit `enter`

2. Create a Kubernetes secret `secret.yaml` by pasting the following into your terminal

```shell
cat > ./secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: runner-oauth-credentials
  labels:
    accountUuid: ${ACCOUNT_UUID}
    runnerUuid: ${RUNNER_UUID}
data:
  oauthClientId: ${BASE64_OAUTH_CLIENT_ID}
  oauthClientSecret: ${BASE64_OAUTH_CLIENT_SECRET}
EOF
```

3. Create a Kubernetes job `job.yaml` by pasting the following into your terminal

```shell
cat > ./job.yaml <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: runner
spec:
  template:
    metadata:
      labels:
        accountUuid: ACCOUNT_UUID
        runnerUuid: RUNNER_UUID
    spec:
      containers:
        - name: bitbucket-k8s-runner
          image: docker-public.packages.atlassian.com/sox/atlassian/bitbucket-pipelines-runner
          env:
            - name: ACCOUNT_UUID
              value: "{${ACCOUNT_UUID}}"
            - name: RUNNER_UUID
              value: "{${RUNNER_UUID}}"
            - name: OAUTH_CLIENT_ID
              valueFrom:
                secretKeyRef:
                  name: runner-oauth-credentials
                  key: oauthClientId
            - name: OAUTH_CLIENT_SECRET
              valueFrom:
                secretKeyRef:
                  name: runner-oauth-credentials
                  key: oauthClientSecret
            - name: WORKING_DIRECTORY
              value: "/tmp"
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: docker-containers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: var-run
              mountPath: /var/run
        - name: docker-in-docker
          image: docker:20.10.7-dind
          securityContext:
            privileged: true
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: docker-containers
              mountPath: /var/lib/docker/containers
            - name: var-run
              mountPath: /var/run
      restartPolicy: OnFailure
      volumes:
        - name: tmp
        - name: docker-containers
        - name: var-run
  backoffLimit: 6
  completions: 1
  parallelism: 1
EOF
```


## Monitor the Job

To monitor the progress of the Bitbucket Runner job, use the following command:

```sh
kubectl logs job/runner -n bb-runner
```

Check the logs to ensure the runner is successfully registered with Bitbucket and is ready to execute pipeline tasks.


## Summary

This guide helps you deploy a Bitbucket Pipeline runner as a Kubernetes job using secure environment variable management. By storing sensitive data in Kubernetes secrets and using them in the runner job, you ensure the safety and security of the credentials used to connect to Bitbucket.