---
title: "Using Tekton for generic automation"
date: 2022-05-01
draft: true
---

I have been looking for a generic automation framework that would let me define
periodic jobs that I want to run. I had looked at a few existing solutions in the space but none
of them really met my needs. Since I already have a OpenShift cluster running in
my home lab I thought I would take Tekton for a spin and see if it could meet my
needs.

<!--more-->

## The problem to solve

In order to get a baseline for evaluating the solutions I settled on a fairly simple automation problem. I wanted to poll a RSS feed for RasberryPi's and alert if something changed and there were RPi4's available for purchase.

(todo add graph of automation solution)

This would test a few different features:

1. Storing persistent state between executions
1. cron-based executions
1. testing the no-code/low-code platform

## Existing solutions

There were a few solutions that came up on my radar over the years that could scratch this particular itch which included n8n, node-RED, and Huginn. I really liked the breath of integrations that came out of the box with all of these solutions but in the end I felt like I was fighting the no-code/low-code solution and that I wanted something slightly higher than straight python but lower than where these solutions were positioned.

## Enter Tekton

Per their website,

> Tekton is a powerful and flexible open-source framework for creating CI/CD systems,
> allowing developers to build, test, and deploy across cloud providers and
> on-premise systems."

Tekton is Kubernetes-native and stores all of it's configuration as Kubernetes Custom Resource Definitions. This made it an extremely compelling option for me since I already am running Kubernetes in the production environment that will run the automation.

Just a little bit about Tekton concepts:

### Tekton Tasks

Tasks are the smallest execution component in Tekton. What is neat about these is that they can be shared either at the cluster or namespace level and there is the [Tekton Hub](https://hub.tekton.dev) where people can share common snippets such as curl or sendmail. Tasks functionally coorrespond to Kubernetes Pods, that is to say that when a Task is instantiated, a Pod will be created by the Tekton controller which has a container for each step in the Task. This means that each step can have it's own container image but that you also have to take special care when sharing data between steps in a Task.

### Tekton Pipelines

The Pipeline is the real foundational component of Tekton which is what defines the sequence of Tasks that will run when the Pipeline is instantiated. The pipeline brings everything together and is the primary resource a user interacts with when using Tekton.

### Instantiating Tasks and Pipelines

Tekton uses a pattern for imperative instantiation that I am seeing becoming more common among Kubernetes operators by defining new custom resources called TaskRun and PipelineRun which reference the Task or Pipeline and is the record that keeps track of the execution.

These objects are eventually garbage collected by a CronJob managed by Tekton.

### Storage as Workspaces

Tekton abstracts shared storage with a Workspace which is exposed by each Task. In the case of a TaskRun a workspace is directly linked with some type of Kubernetes Volume but in the case of a PipelineRun the Task workspaces are mapped to Pipeline workspaces which are then linked to a Kubernetes Volume.

Since every step in a Task runs in it's own container and every Task in a Pipeline runs a separate Kubernetes Pod if we want to share data between steps and tasks we will need to use something like a PersistentVolumeClaim.

## Automating with Tekton

### Python Task

In order to process the XML RSS feed I realized I would need to write some Python but I was able to get away with just using the standard library for my minor parsing needs. I started writing a Task to contain my steps using Python to parse the XML until I ran across an example in the documentation to inline a Task in a Pipeline. Since this particular logic wouldn't be reused anywhere I went with that method.

### Getting the XML

Since I did not want to `pip install` anything if possible I decided to separate out the step of actually obtaining the XML feed data.

I first started by attempting to use the existing Tekton Hub Task for curl but ran into permission problems with the user that the curlimages/curl image was set to run as and my PersistentVolumeClaim that I was using for storage and settled on just using the Red Hat UBI images which included curl.

### Storage

Tying all of the steps together was a pre-allocated PVC. The PipelineRun instantiation is what links the Pipeline workspace with the Kubernetes Volume.

{{< highlight yaml >}}

{{< /highlight >}}

### CronJob

In order to kick off the PipelineRun every so often I used a CronJob. Some neat features of this CronJob:

- I was able to use the cli image that is already present in the OpenShift cluster
- When creating a PipelineRun I let Kubernetes come up with a unique hash by using `.metadata.generateName`

{{< highlight yaml >}}
kind: CronJob
apiVersion: batch/v1beta1
metadata:
  name: rpilocator-cronjob
spec:
  schedule: '*/30 * * * *'
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      backoffLimit: 3
      template:
        spec:
          containers:
            - name: cli
              image: >-
                image-registry.openshift-image-registry.svc:5000/openshift/cli:latest
              args:
                - /bin/bash
                - '-c'
                - |
                  oc create -n automation -f - << EOF
                  apiVersion: tekton.dev/v1beta1
                  kind: PipelineRun
                  metadata:
                    generateName: rpilocator-
                  spec:
                    pipelineRef:
                      name: rpilocator
                    workspaces:
                    - name: work
                      persistentVolumeClaim:
                        claimName: rpilocator
                  EOF
          restartPolicy: OnFailure
          serviceAccountName: rpilocator-cron
          serviceAccount: rpilocator-cron
      ttlSecondsAfterFinished: 3600
  successfulJobsHistoryLimit: 2
  failedJobsHistoryLimit: 1

{{< /highlight >}}
