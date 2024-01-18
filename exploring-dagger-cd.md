---
title: "Exploring Dagger: Continuous Deployment with less YAML and more Code"
date: 2023-12-01T00:00:00-03:00
tags: ["go", "dagger", "ci/cd", "exploring-dagger"]
categories: ["infrastructure"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
disableHLJS: false
disableShare: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
UseHugoToc: true
cover:
    alt: "Exploring Dagger: Continuous Deployment with less YAML and more Code"
    relative: false
    hidden: true
editPost:
    URL: "https://github.com/matipan/matipan.github.io/tree/main/content"
    Text: "Suggest Changes"
    appendFilePath: true
---

This is the third post in a [series of blog posts](/tags/exploring-dagger/) that look at Dagger from different perspectives. In this post we pick up from where we left off in the [last post](/posts/exploring-dagger-streamlining-ci-cd-pipelines-with-code) and we will focus on building a continuous deployment process for deploying applications in a mid-sized company (meaning the process needs to be used for many services). As we did before, we will first build this process using "traditional" tools that are very frequently used for this purpose and we will compare it to building it using Dagger. The goal of this series of blog posts is to understand where Dagger fits in the industry and where it does not.

**NOTE**: Dagger is in very active development so by the time you read this blog post some things might have changed.

## Background

I'll start off with a disclaimer: deploying applications is done in a dramatically different way on each company. In the past two blog posts we looked at solving problems from an SRE and developer perspective that are quite commonly solved in the same way, however, when it comes to deploying applications it seems that we as an industry can't agree on a generalized approach. You can see this not only by looking at what companies actually do but also at how many companies and projects exist that attempt to solve this problem, each in their own opinionated way. Some companies built custom platforms, some use "raw" Kubernetes, some use ECS, some use Nomad and some still deploy directly to a host. In this article I wanted the comparisson to be as realistic as possible, but given the vast number of tools and approaches available it seems that I will have to pick one and compare against that one. In my own experience and the conversations I've had with folks from other companies, I found the approach of using Kubernetes + Helm or Kustomize to be the most common one. The rationale behind this approach is that it provides a good level of flexibility to developers while still abstracting away some parts of Kubernetes. Helm (or Kustomize) are often already being used by SREs to install dependencies on the cluster, so carrying on that trend to also deploy internal services seems to be a "natural" evolution. Let's go over what our base infrastructure will look like with this in mind:
* **Deployment**: services are deployed in a single k8s cluster. Developers own the definition of their application, including env variables, secrets, routing rules and auto scaling policies.
* **Routing**: traffic routing is done inside kubernetes using Traefik as an Ingress controller.
* **Static infrastructure**: [static infrastructure](/posts/how-services-are-provisioned-and-deployed-at-lemoncash/#concepts) like databases, queues, caches, domains and others are managed outside of k8s and controlled by an [IaC repository](/posts/exploring-dagger-building-a-ci-cd-pipeline-for-iac/).
* **Exposing traffic**: services cannot automagically expose their endpoints directly. Instead we have a gateway that controls rate limiting, authentication and authorization. We will not cover how to solve this since this is where I find the approach between companies vary significantly.
* **Observability**: services are expected to expose an OpenMetrics endpoint that our agent will scrape. We will not cover this in much detail since it is not very relevant for the CD process.

Like we mentioned before, we want to give developers a good level of flexibility and as much ownership as possible of the deployment process. But we don't want them to have to be k8s experts so we want to build abstractions that attempt to find this balance. The level of configuration we initially want to give would allow developers to:
* Define Auto Scaling policies according to a predefined set of criterias.
* Define the deployment strategy for their service.
* Define the path used for routing traffic to it from outside the k8s cluster.
* Define custom variables, secrets and mount configurations to the filesystem.
* Define custom health checks.

We will have a few conventions to provide default values for each of the items above. Such as using the name of the service for traffic routing, the `/health` endpoint for health checks and port 8080 for exposing HTTP traffic.

## Using Helm with custom charts

If you are deploying your company's services on k8s and you want to abstract it away from developers then it is quite common to default to using [Helm](https://helm.sh/) since it is often already being used as a package manager to install dependencies such as observability agents, ingress controllers and others. This is an approach that I've seen in many companies and that I've myself have built in the past. This is not necessarily the best approach out there and I know many companies have widely different ways of solving this problem. Many don't use Kubernetes and instead deploy on more managed tools. If that is your case, stick around anyway! You can jump straight into the [Dagger section](#using-dagger-with-custom-modules) and learn a bit about this new tool.

I chose to use Helm for this comparison article mainly because of popularity and familiarity. In order to understand how Dagger can be used to solve this type of problems my brain needs to compare it to something it already knows.

So, lets get started. We mentioned previously what we want to initially expose to developers. Quick refresher:
* Define Auto Scaling policies according to a predefined set of criteria.
* Define the deployment strategy for their service.
* Define the path used for routing traffic to it from outside the k8s cluster.
* Define custom variables, secrets and mount configurations to the filesystem.
* Define custom health checks.

Based on these requirements we first need to understand which k8s resources we are going to need:
* **Deployment**: developers will have to specify the image and can potentially specify deployment strategies, custom health checks, etc.
* **HorizontalPodAutoscaler**: developers can optionally use HPA to automatically scale their service.
* **Service**: traffic inside the cluster will be routed using a ClusterIP service.
* **Ingress**: traffic outside the cluster (inside our network and potentially outside of it) will be routed using Traefik as an ingress controller.

With that in mind let's go directly into building the YAML file that will be exposed to developers. This will be the "interface" that developers will have for configuring their service:
```yaml
image:
  ref: "registry/repo:tag"
  pullPolicy: Always

labels: {}

resources:
  limits:
    memory: 500Mi
  requests:
    cpu: 500m
    memory: 500Mi

deployment:
  replicaCount: 1
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  terminationGracePeriodSeconds: 30
  restartPolicy: Always

service:
 port: 80
 targetPort: 8080
 protocol: TCP
 publicPort:
   port: 80
   targetPort: 80
   name: http
 healthCheckPort:
   port: 80
   targetPort: 8080

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80

ingress:
  enabled: false

env: {}

secrets: {}
```

We could restructure this YAML to express our dependencies in a way that is not tied to Kubernetes's resources. But for today we are keeping things as straightforward as possible. While changing this could make the interface slightly better, it won't that big of an impact from a developer perspective.

The way we transform this YAML into k8s resources with Helm is quite simple (but painful 👀). We need to write more YAML files that use Go templates to fill in the values that were specified in the `values.yaml` file that developers wrote. It's YAML all the way. I'll show only the `deployment.yaml` file for brevity but you can see the entire chart in [this repo]():
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "servicer.fullname" . }}
  namespace: services
  labels:
    app.kubernetes.io/name: {{ include "servicer.name" . }}
    helm.sh/chart: {{ include "servicer.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
    app: {{ include "servicer.name" . }}
    {{- if not .Values.labels.service }}
    service: {{ include "servicer.name" . }}
    {{- end }}
    {{- with .Values.labels }}
      {{- toYaml . | nindent 4 }}
    {{- end }}
spec:
  # If autoscaling is enabled then the replica count should never be managed
  # by this field. Instead, users should edit the definition of autoscaling 
  # and specify min replicas and max replicas.
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.deployment.replicaCount }}
  {{- end }}
  selector:
    matchLabels:  
      app.kubernetes.io/name: {{ include "servicer.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  strategy:
    type:  {{ .Values.deployment.strategy.type }}
    {{- if .Values.deployment.strategy.rollingUpdate }}
    rollingUpdate:
      maxUnavailable: {{ .Values.deployment.strategy.rollingUpdate.maxUnavailable }}
      maxSurge: {{ .Values.deployment.strategy.rollingUpdate.maxSurge }}
    {{- end }}
  template:
    metadata:
      labels:
        {{- if not .Values.labels.service }}
        service: {{ include "servicer.name" . }}
        {{- end }}
        app.kubernetes.io/name: {{ include "servicer.name" . }}
        app: {{ include "servicer.name" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
        {{- with .Values.labels }}
          {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      containers:
        - name: {{ include "servicer.name" . }}
          image: "{{ .Values.image.ref }}"
          imagePullPolicy: {{ .Values.image.pullPolicy | default "IfNotPresent" }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort | default 8080 }}
              protocol: {{ .Values.service.protocol | default "http" }}
          livenessProbe:
            httpGet:
              path: {{ .Values.service.liveness.path | default "/health" }}
              port: {{ .Values.service.liveness.port | default .Values.service.targetPort }}
              {{- if .Values.service.liveness.headers }}
              httpHeaders:
              {{- range $key, $value := .Values.service.liveness.headers }}
              - name: {{ $key }}
                value: {{ $value | quote }}
              {{- end }}
              {{- end }}
            initialDelaySeconds: {{ .Values.service.liveness.initialDelaySeconds | default 30 }}
            periodSeconds: {{ .Values.service.liveness.periodSeconds | default 10 }}
            failureThreshold: {{ .Values.service.liveness.failureThreshold | default 3 }}
            timeoutSeconds: {{ .Values.service.liveness.timeoutSeconds | default 1 }}
          readinessProbe:
            httpGet:
              path: {{ .Values.service.readiness.path | default "/health" }}
              port: {{ .Values.service.readiness.port | default .Values.service.targetPort }}
              {{- if .Values.service.readiness.headers }}
              httpHeaders:
              {{- range $key, $value := .Values.service.readiness.headers }}
              - name: {{ $key }}
                value: {{ $value | quote }}
              {{- end }}
              {{- end }}
            initialDelaySeconds: {{ .Values.service.readiness.initialDelaySeconds | default 30 }}
            periodSeconds: {{ .Values.service.readiness.periodSeconds | default 10 }}
            failureThreshold: {{ .Values.service.readiness.failureThreshold | default 3 }}
            timeoutSeconds: {{ .Values.service.readiness.timeoutSeconds | default 1 }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- if or $.Values.env $.Values.secrets }}
          env:
            {{- if $.Values.env  }}
            {{- range $key, $value := $.Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
            {{- end }}
            {{- if $.Values.secrets  }}
            {{- range $key, $value := $.Values.secrets }}
            - name: {{ $key }}
              valueFrom: {{ toYaml $value | nindent 16 }}
            {{- end }}
            {{- end }}
          {{- end }}
```

Let's go over a few of the important bits of this Deployment spec so that we can better understand what it means to build something like this:

**Auto Scaling**  
If developers enable auto-scaling we should not set the `replica` field since it will conflict with whatever the HPA controller wants to do. So, we use the value `.Values.autoscaling.enabled` and only set replicas when it is not specified:

```yaml
  # If autoscaling is enabled then the replica count should never be managed
  # by this field. Instead, users should edit the definition of autoscaling 
  # and specify min replicas and max replicas.
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.deployment.replicaCount }}
  {{- end }}
```

**Health Checks**  
We want developers to control how their service is checked for both readiness and liveness. However, we want to have some conventions that developers can follow so that they don't have to specify health checks. In the code below we configure the value of `path`, `port` and others with what was specified but if nothing was specified we default to values like `/health` and the port of the service (which by default would be 8080):

```yaml
  livenessProbe:
    httpGet:
      path: {{ .Values.service.liveness.path | default "/health" }}
      port: {{ .Values.service.liveness.port | default .Values.service.targetPort }}
      {{- if .Values.service.liveness.headers }}
      httpHeaders:
      {{- range $key, $value := .Values.service.liveness.headers }}
      - name: {{ $key }}
        value: {{ $value | quote }}
      {{- end }}
      {{- end }}
    initialDelaySeconds: {{ .Values.service.liveness.initialDelaySeconds | default 30 }}
    periodSeconds: {{ .Values.service.liveness.periodSeconds | default 10 }}
    failureThreshold: {{ .Values.service.liveness.failureThreshold | default 3 }}
    timeoutSeconds: {{ .Values.service.liveness.timeoutSeconds | default 1 }}
```

**Env variables and secrets**  
Environment variables and secrets can either be an inline value or get the value from a specific resource, for example the IP of the host:
```yaml
- name: HOST_IP
  valueFrom:
      fieldRef:
          fieldPath: status.hostIP
```

To manage this we have to iterate over the specified values and set them accordingly. For the case of referenced value we have to do some "fancy" stuff and render the YAML directly: 
```yaml
  env:
    {{- if $.Values.env  }}
    {{- range $key, $value := $.Values.env }}
    - name: {{ $key }}
      value: {{ $value | quote }}
    {{- end }}
    {{- end }}
    {{- if $.Values.secrets  }}
    {{- range $key, $value := $.Values.secrets }}
    - name: {{ $key }}
      valueFrom: {{ toYaml $value | nindent 16 }}
    {{- end }}
    {{- end }}
  {{- end }}
```

With this developers can specify both inline and referenced values like so:
```yaml
env:
  ENV: prod
  DB_URL: ...

secrets:
  HOST_IP:
    fieldRef:
      fieldPath: status.hostIP
```

In the case of ingress, we know that not all services require it. For this reason the entire resource is behind a condition that indicates if the service needs it or not. If it does, we use a convention of using the name of the service for routing traffic to it:
```yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: services
  name: {{ include "servicer.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "servicer.name" . }}
    helm.sh/chart: {{ include "servicer.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  rules:
    - http:
        paths:
          {{- $name := include "servicer.fullname" . }}
          - path: {{ printf "/%s" $name }}
            pathType: Prefix
            backend:
              service:
                name: {{ include "servicer.fullname" . }}
                port:
                  name: http
{{- end }}
```

If you want to see the entire chart you can go to this [repo]().

### Building our CD workflow

### Thoughts

It is not a secret that I don't particularly like the experience of building custom charts for deploying services:

{{<tweet user="matiaspan26" id="1730924500055646583">}}

I find Helm quite useful as a package manager to install applications such as Traefik, Redis or other more static components. But when it comes to building custom charts for deploying services, I find the experience sucks. Unlike applications such as Traefik, this custom charts that deploy services are iterated on quite frequently. New requirements come in and tools and conventions often change. I find the experience of developing templates that use values loosely referred in a values.yaml file very annoying and error prone.

## Using Dagger with custom modules

### Building our CD workflow

### Thoughts

## Extensibility

Adding the ability to add sidecars? Or maybe even re-define completely the list of containers that are deployed and how they are actually deployed

### With Helm

### With Dagger

## Conclusion

