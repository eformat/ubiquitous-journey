## Help me 

### Not automated yet ...
 - [ ]  Binding the  `nexus-sonatype-nexus` Service Account to the Role view to be able to read ConfigMaps when creating the nexus repos
```
oc adm policy add-role-to-user view -z nexus-sonatype-nexus
```

 - [ ]  Create ArgoCD token in UI or on CMD Line. Update config map to give `apiKey` capability to the admin account. Then generate token (ui or cli). Use `basic-auth` secret type with `username: token` and `password: aaa.bbb.ccc` as Jenkins will give you a JSON object if you just use `opaque`. This way you get the vars `ARGOCD_CREDS_PSW` and you're away. Note ArgoCD Admin passwd is now stored in a secret called 

```
$ oc edit cm argocd-cm

data:
  accounts.admin: apiKey

$ argocd account generate-token --account admin 
```

- [ ] `dummy-sa` should become `jenkins` :wink:

- [ ] Generate GITHUB personal access token or whatever to be able to push git updates as part of jenkins workflow


## Helm Charts, Versioning, ArgoCD Applications

Example `Jenkinsfile` - https://github.com/eformat/pet-battle-api/blob/test/jenkins/Jenkinsfile

- [ ] As a developer, i wish to use Jenkins Multibranch Pipeline plugin for my gitops workflow.
- [ ] As a developer, i wish to practice Trunk Based development with short lived feature or fix branches within my application teams.

OpenShift4 is evolving support for helm3. Unfortunately, there is no current way to store Charts in the internal registry (other than using the helm operator approach). Quay has experimental support for helm charts.

We currently have argocd managing helm charts. Helm does not mandate the use of helm chart repositories i.e. it is possible to reference a helm Chart directly from git.

Helm Best practices suggest using a chart repository - especially when going between environments is advisable. It is worth comparing the approaches.

- [ ] // FIXME - We need an updated helm chart nexus version (current 3.19.1-01 -> 3.22.1-02), newer version (which breaks with kube plugin for pwd). Note the current nexus also does not deploy the `labs-static` raw from ConfigMap as it should.

So, an example workflow for `pet-battle-api` uses a helm chart repository in nexus as follows:

1. perpare environment - sets env variables based on branch
2. creates argocd app - uses inline yaml for now rather than argocd cli, as we need ignoreDifferences set
3. builds the app - builds a quarkus thin jar using a mvn slave in this case (mvn 3.6, jdk11) and stores the resulting built artifacts in the nexus raw repo as tgz
4. openshift binary build to containerize application - uses ubi8, jdk11 as base image, combines thin jar, libs (native build could also be used but java.io/image libs not part or quarkus yet)
5. git commits chart and values with updates versions - git commit to branch
6. uploads the helm chart to nexus helm repository
7. syncs the argocd application using the updated helm chart

To support multiple branches being built and deployed in the same Namespaces, heml Charts need to be able to run in the same Namespace using helm `Release`.

For example where `foo` is the release name:
```bash
helm template --name-template=foo ...
helm install foo ...
```
A real life example of two Releases of the same chart in the same namespace can be tested using the cli:
```bash
helm template --name-template=foo -f chart/values.yaml chart | oc apply -f- --dry-run --validate
helm template --name-template=bar -f chart/values.yaml chart | oc apply -f- --dry-run --validate
```

In argocd the Release name is specified as:
```yaml
  spec:
    helm:
      releaseName: {{ $app.name }}
```

In Helm, there is an (optional) `appVersion` and a chart version - so the app and chart can be versioned separately if desired.
```yaml
Chart:
  version: 0.0.1       # the chart specific version
  appVersion: latest   # the application version
```

I'm currently playing with versioning. It seems most obvious to change the `appVersion` when we build a new image.

The Chart version - could remain static, although we have have seen chart cache issues, currently am hacking this:

- [ ] // FIXME ${SEM_VER} generation is required for Chart::version, appVersion can be a string

I'm using the `appVersion` and linking it to the main application image version, via an image stream tag name:
```yaml
  tags:
    - name: {{ .Chart.AppVersion }}
      from:
        kind: DockerImage
        name: {{ .Values.image_repository }}/{{ .Values.image_namespace }}/{{ .Values.image_name }}:{{ .Chart.AppVersion }}
      importPolicy: {}
```

This means we can support an image stream per branch, by using the `template` function for the `fullname` in our kube objects:
```yaml
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
```
And:
```yaml
kind: ImageStream
metadata:
  name: {{ include "pet-battle-api.fullname" . }}
```

Other Things:
- [ ] Think some more about supporting umbrealla Charts (have chart dependencies) that may have multiple image versions, especially where Release is not adhered to in naming kube objects (namespace isolation only in these cases ?)

Broken Things:
- [ ] Chart SEMVER in Jenkinsfile
- [ ] Nexus image & helm support if nexus helm repository in use - https://github.com/sonatype-nexus-community/nexus-kubernetes-openshift/issues/46
- [ ] Custom java mvn slave - is custom with mvn3.6, jdk1.11 cause there is no rh-maven36 in centos, base image - UBI8 has mvn3.5.4
- [ ] ArgoCD RBAC - apiKey for admin user