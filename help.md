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

- [ ] As a developer, i wish to use Jenkins Multibranch Pipeline plugin for my gitops workflow.
- [ ] As a developer, i wish to practice Trunk Based development with short lived feature or fix branches within my application teams.

To support developing and building application Charts in the same namespace, we can use the helm `Release` name to map to our git branch. For example where `foo` is the release name:
```bash
helm template --name-template=foo ...
helm install foo ...
```
A real life example of two Releases of the same chart in the same namespace:
```bash
helm template --name-template=foo -f chart/values.yaml chart | oc apply -f- --dry-run --validate
helm template --name-template=bar -f chart/values.yaml chart | oc apply -f- --dry-run --validate
```

In argocd the release name is specified as:
```yaml
  spec:
    helm:
      releaseName: {{ $app.name }}
```

In Helm, there is an (optional) `appVersion` and a chart version - so they app and chart can be versioned separately.
```yaml
Chart:
  version: 0.0.1       # the chart specific version
  appVersion: latest   # the application version
```

We will use the `appVersion` and link it to the main application version, preferably the image stream tag name:
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

Other things:
- [ ] Think some more about supporting umbrealla Charts (have chart dependencies) that may have multiple image versions, especially where Release is not adhered to in naming kube objects (namespace isolation only in these cases ?)