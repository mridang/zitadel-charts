[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/zitadel)](https://artifacthub.io/packages/search?repo=zitadel)

# Zitadel

## A Better Identity and Access Management Solution

Identity infrastructure, simplified for you.

Learn more about Zitadel by checking out the [source repository on GitHub](https://github.com/zitadel/zitadel)

## What's in the Chart

By default, this chart installs a highly available Zitadel deployment.

The chart deploys a Zitadel init job, a Zitadel setup job and a Zitadel deployment.
By default, the execution order is orchestrated using Helm hooks on installations and upgrades.

## Install the Chart

The easiest way to deploy a Helm release for Zitadel is by following the [Insecure Postgres Example](examples/1-postgres-insecure/README.md).
For more sofisticated production-ready configurations, follow one of the following examples:

- [Secure Postgres Example](examples/2-postgres-secure/README.md)
- [Referenced Secrets Example](examples/3-referenced-secrets/README.md)
- [Machine User Setup Example](examples/4-machine-user/README.md)
- [Internal TLS Example](examples/5-internal-tls/README.md)

All the configurations from the examples above are guaranteed to work, because they are directly used in automatic acceptance tests.

## Upgrade From V8 to V9

The v9 charts default Zitadel and login versions reference [Zitadel v4](https://github.com/zitadel/zitadel/releases/tag/v4.0.0).

### Don’t Switch to the New Login Deployment

Use `login.enabled: false` to omit deploying the new login.

### Switch to the New Login Deployment

By default, a new deployment for the login v2 is configured and created.
For new installations, the setup job automatically creates a user of type machine with role `IAM_LOGIN_CLIENT`.
It writes the users personal access token into a Kubernetes secret which is then mounted into the login pods.

For existing installations, the setup job doesn't create this login client user.
Therefore, the Kubernetes secret has to be created manually before upgrading to v9:

1. Create a user of type machine
2. Make the user an instance administrator with role `IAM_LOGIN_CLIENT`
3. Create a personal access token for the user
4. Create a secret with that token: `kubectl --namespace <my-namespace> create secret generic login-client --from-file=pat=<my-local-path-to-the-downloaded-pat-file>`

To make the login externally accessible, you need to route traffic with the path prefix `/ui/v2/login` to the login service.
If you use an ingress controller, you can enable the login ingress with `login.ingress.enabled: true`

> [!CAUTION]
> Don't Lock Yourself Out of Your Instance
> Before you change your Zitadel configuration, we highly recommend you to create a service user with a personal access token (PAT) and the IAM_OWNER role.
> In case something breaks, you can use this PAT to revert your changes or fix the configuration so you can use a login UI again.

To actually use the new login, enable the loginV2 feature on the instance.
Leave the base URI empty to use the default or explicitly configure it to `/ui/v2/login`.
If you enable this feature, the login will be used for every application configured in your Zitadel instance.

### Other Breaking Changes

- Default Traefik and NGINX annotations for internal unencrypted HTTP/2 traffic to the Zitadel pods are added.
- The default value `localhost` is removed from the Zitadel ingresses `host` field. Instead, the `host` fields for the Zitadel and login ingresses default to `zitadel.configmapConfig.ExternalDomain`.
- The following Kubernetes versions are tested:
  - v1.33.1
  - v1.32.5
  - v1.31.9
  - v1.30.13

## Upgrade from v7

> [!WARNING] The chart version 8 doesn't get updates to the default Zitadel version anymore as this might break environments that use CockroachDB.
> Please set the version explicitly using the appVersion variable if you need a newer Zitadel version.
> The upcoming version 9 will include the latest Zitadel version by default (Zitadel v3).

The default Zitadel version is now >= v2.55.
[This requires Cockroach DB to be at >= v23.2](https://zitadel.com/docs/support/advisory/a10009)
If you are using an older version of Cockroach DB, please upgrade it before upgrading Zitadel.

Note that in order to upgrade cockroach, you should not jump minor versions.
For example:

```bash
# install Cockroach DB v23.1.14
helm upgrade db cockroachdb/cockroachdb --version 11.2.4 --reuse-values
# install Cockroach DB v23.2.5
helm upgrade db cockroachdb/cockroachdb --version 12.0.5 --reuse-values
# install Cockroach DB v24.1.1
helm upgrade db cockroachdb/cockroachdb --version 13.0.1 --reuse-values
# install Zitadel v2.55.0
helm upgrade my-zitadel zitadel/zitadel --version 8.0.0 --reuse-values
```

Please refer to the docs by Cockroach Labs. The Zitadel tests run against the [official CockroachDB chart](https://artifacthub.io/packages/helm/cockroachdb/cockroachdb).

(Credits to @panapol-p and @kleberbaum :pray:)

## Upgrade from v6

- Now, you have the flexibility to define resource requests and limits separately for the machineKeyWriter,
  distinct from the setupJob.
  If you don't specify resource requests and limits for the machineKeyWriter,
  it will automatically inherit the values used by the setupJob.

- To maintain consistency in the structure of the values.yaml file, certain properties have been renamed.
  If you are using any of the following properties, kindly review the updated names and adjust the values accordingly:

  | Old Value                                   | New Value                                    |
  |---------------------------------------------|----------------------------------------------|
  | `setupJob.machinekeyWriterImage.repository` | `setupJob.machinekeyWriter.image.repository` |
  | `setupJob.machinekeyWriterImage.tag`        | `setupJob.machinekeyWriter.image.tag`        |

## Upgrade from v5

- CockroachDB is not in the default configuration anymore.
  If you use CockroachDB, please check the host and ssl mode in your Zitadel Database configuration section.

- The properties for database certificates are renamed and the defaults are removed.
  If you use one of the following properties, please check the new names and set the values accordingly:

  | Old Value                      | New Value                     |
  |--------------------------------|-------------------------------|
  | `zitadel.dbSslRootCrt`         | `zitadel.dbSslCaCrt`          |
  | `zitadel.dbSslRootCrtSecret`   | `zitadel.dbSslCaCrtSecret`    |
  | `zitadel.dbSslClientCrtSecret` | `zitadel.dbSslAdminCrtSecret` |
  | `-`                            | `zitadel.dbSslUserCrtSecret`  |

## Uninstalling the Chart

The Zitadel chart uses Helm hooks,
[which are not garbage collected by helm uninstall, yet](https://helm.sh/docs/topics/charts_hooks/#hook-resources-are-not-managed-with-corresponding-releases).
Therefore, to also remove hooks installed by the Zitadel Helm chart,
delete them manually:

```bash
helm uninstall my-zitadel
for k8sresourcetype in job configmap secret rolebinding role serviceaccount; do
    kubectl delete $k8sresourcetype --selector app.kubernetes.io/name=zitadel,app.kubernetes.io/managed-by=Helm
done
```

## Configuration

### Values

<!-- render.chart.valuesTable -->
| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | object | `{}` |  |
| annotations | object | `{}` |  |
| configMap.annotations."helm.sh/hook" | string | `"pre-install,pre-upgrade"` |  |
| configMap.annotations."helm.sh/hook-delete-policy" | string | `"before-hook-creation"` |  |
| configMap.annotations."helm.sh/hook-weight" | string | `"0"` |  |
| env | list | `[]` |  |
| envVarsSecret | string | `""` |  |
| extraContainers | list | `[]` |  |
| extraManifests | list | `[]` |  |
| extraVolumeMounts | list | `[]` |  |
| extraVolumes | list | `[]` |  |
| fullnameOverride | string | `""` |  |
| image.pullPolicy | string | `"IfNotPresent"` |  |
| image.repository | string | `"ghcr.io/zitadel/zitadel"` |  |
| image.tag | string | `""` |  |
| imagePullSecrets | list | `[]` |  |
| ingress.annotations."nginx.ingress.kubernetes.io/backend-protocol" | string | `"GRPC"` |  |
| ingress.className | string | `""` |  |
| ingress.enabled | bool | `false` |  |
| ingress.hosts[0].paths[0].path | string | `"/"` |  |
| ingress.hosts[0].paths[0].pathType | string | `"Prefix"` |  |
| ingress.tls | list | `[]` |  |
| initJob.activeDeadlineSeconds | int | `300` |  |
| initJob.annotations."helm.sh/hook" | string | `"pre-install,pre-upgrade"` |  |
| initJob.annotations."helm.sh/hook-delete-policy" | string | `"before-hook-creation"` |  |
| initJob.annotations."helm.sh/hook-weight" | string | `"1"` |  |
| initJob.backoffLimit | int | `5` |  |
| initJob.command | string | `""` |  |
| initJob.enabled | bool | `true` |  |
| initJob.extraContainers | list | `[]` |  |
| initJob.initContainers | list | `[]` |  |
| initJob.podAdditionalLabels | object | `{}` |  |
| initJob.podAnnotations | object | `{}` |  |
| initJob.resources | object | `{}` |  |
| livenessProbe.enabled | bool | `true` |  |
| livenessProbe.failureThreshold | int | `3` |  |
| livenessProbe.initialDelaySeconds | int | `0` |  |
| livenessProbe.periodSeconds | int | `5` |  |
| login.affinity | object | `{}` |  |
| login.configMap.annotations."helm.sh/hook" | string | `"pre-install,pre-upgrade"` |  |
| login.configMap.annotations."helm.sh/hook-delete-policy" | string | `"before-hook-creation"` |  |
| login.configMap.annotations."helm.sh/hook-weight" | string | `"0"` |  |
| login.customConfigmapConfig | string | `nil` |  |
| login.enabled | bool | `true` |  |
| login.env | list | `[]` |  |
| login.extraContainers | list | `[]` |  |
| login.extraVolumeMounts[0].mountPath | string | `"/login-client"` |  |
| login.extraVolumeMounts[0].name | string | `"login-client"` |  |
| login.extraVolumeMounts[0].readOnly | bool | `true` |  |
| login.extraVolumes[0].name | string | `"login-client"` |  |
| login.extraVolumes[0].secret.defaultMode | int | `444` |  |
| login.extraVolumes[0].secret.secretName | string | `"login-client"` |  |
| login.fullnameOverride | string | `""` |  |
| login.image.pullPolicy | string | `"IfNotPresent"` |  |
| login.image.repository | string | `"ghcr.io/zitadel/zitadel-login"` |  |
| login.image.tag | string | `""` |  |
| login.imagePullSecrets | list | `[]` |  |
| login.ingress.annotations | object | `{}` |  |
| login.ingress.className | string | `""` |  |
| login.ingress.enabled | bool | `false` |  |
| login.ingress.hosts[0].paths[0].path | string | `"/ui/v2/login"` |  |
| login.ingress.hosts[0].paths[0].pathType | string | `"Prefix"` |  |
| login.ingress.tls | list | `[]` |  |
| login.initContainers | list | `[]` |  |
| login.livenessProbe.enabled | bool | `true` |  |
| login.livenessProbe.failureThreshold | int | `3` |  |
| login.livenessProbe.initialDelaySeconds | int | `0` |  |
| login.livenessProbe.periodSeconds | int | `5` |  |
| login.loginClientSecretPrefix | string | `nil` |  |
| login.nameOverride | string | `""` |  |
| login.nodeSelector | object | `{}` |  |
| login.podAdditionalLabels | object | `{}` |  |
| login.podAnnotations | object | `{}` |  |
| login.podSecurityContext.fsGroup | int | `1000` |  |
| login.podSecurityContext.runAsNonRoot | bool | `true` |  |
| login.podSecurityContext.runAsUser | int | `1000` |  |
| login.readinessProbe.enabled | bool | `true` |  |
| login.readinessProbe.failureThreshold | int | `3` |  |
| login.readinessProbe.initialDelaySeconds | int | `0` |  |
| login.readinessProbe.periodSeconds | int | `5` |  |
| login.replicaCount | int | `3` |  |
| login.resources | object | `{}` |  |
| login.revisionHistoryLimit | int | `10` |  |
| login.securityContext.privileged | bool | `false` |  |
| login.securityContext.readOnlyRootFilesystem | bool | `true` |  |
| login.securityContext.runAsNonRoot | bool | `true` |  |
| login.securityContext.runAsUser | int | `1000` |  |
| login.service.annotations | object | `{}` |  |
| login.service.appProtocol | string | `"kubernetes.io/http"` |  |
| login.service.clusterIP | string | `""` |  |
| login.service.externalTrafficPolicy | string | `""` |  |
| login.service.labels | object | `{}` |  |
| login.service.port | int | `3000` |  |
| login.service.protocol | string | `"http"` |  |
| login.service.scheme | string | `"HTTP"` |  |
| login.service.type | string | `"ClusterIP"` |  |
| login.serviceAccount.annotations."helm.sh/hook" | string | `"pre-install,pre-upgrade"` |  |
| login.serviceAccount.annotations."helm.sh/hook-delete-policy" | string | `"before-hook-creation"` |  |
| login.serviceAccount.annotations."helm.sh/hook-weight" | string | `"0"` |  |
| login.serviceAccount.create | bool | `true` |  |
| login.serviceAccount.name | string | `""` |  |
| login.startupProbe.enabled | bool | `false` |  |
| login.startupProbe.failureThreshold | int | `30` |  |
| login.startupProbe.periodSeconds | int | `1` |  |
| login.tolerations | list | `[]` |  |
| login.topologySpreadConstraints | list | `[]` |  |
| metrics.enabled | bool | `false` |  |
| metrics.serviceMonitor.enabled | bool | `false` |  |
| metrics.serviceMonitor.honorLabels | bool | `false` |  |
| metrics.serviceMonitor.honorTimestamps | bool | `true` |  |
| nameOverride | string | `""` |  |
| nodeSelector | object | `{}` |  |
| pdb.annotations | object | `{}` |  |
| pdb.enabled | bool | `false` |  |
| pdb.minAvailable | int | `1` |  |
| podAdditionalLabels | object | `{}` |  |
| podAnnotations | object | `{}` |  |
| podSecurityContext.fsGroup | int | `1000` |  |
| podSecurityContext.runAsNonRoot | bool | `true` |  |
| podSecurityContext.runAsUser | int | `1000` |  |
| readinessProbe.enabled | bool | `true` |  |
| readinessProbe.failureThreshold | int | `3` |  |
| readinessProbe.initialDelaySeconds | int | `0` |  |
| readinessProbe.periodSeconds | int | `5` |  |
| replicaCount | int | `3` |  |
| resources | object | `{}` |  |
| securityContext.privileged | bool | `false` |  |
| securityContext.readOnlyRootFilesystem | bool | `true` |  |
| securityContext.runAsNonRoot | bool | `true` |  |
| securityContext.runAsUser | int | `1000` |  |
| service.annotations."traefik.ingress.kubernetes.io/service.serversscheme" | string | `"h2c"` |  |
| service.appProtocol | string | `"kubernetes.io/h2c"` |  |
| service.clusterIP | string | `""` |  |
| service.externalTrafficPolicy | string | `""` |  |
| service.labels | object | `{}` |  |
| service.port | int | `8080` |  |
| service.protocol | string | `"http2"` |  |
| service.scheme | string | `"HTTP"` |  |
| service.type | string | `"ClusterIP"` |  |
| serviceAccount.annotations."helm.sh/hook" | string | `"pre-install,pre-upgrade"` |  |
| serviceAccount.annotations."helm.sh/hook-delete-policy" | string | `"before-hook-creation"` |  |
| serviceAccount.annotations."helm.sh/hook-weight" | string | `"0"` |  |
| serviceAccount.create | bool | `true` |  |
| serviceAccount.name | string | `""` |  |
| setupJob.activeDeadlineSeconds | int | `300` |  |
| setupJob.additionalArgs[0] | string | `"--init-projections=true"` |  |
| setupJob.annotations."helm.sh/hook" | string | `"pre-install,pre-upgrade"` |  |
| setupJob.annotations."helm.sh/hook-delete-policy" | string | `"before-hook-creation"` |  |
| setupJob.annotations."helm.sh/hook-weight" | string | `"2"` |  |
| setupJob.backoffLimit | int | `5` |  |
| setupJob.extraContainers | list | `[]` |  |
| setupJob.initContainers | list | `[]` |  |
| setupJob.machinekeyWriter.image.repository | string | `"bitnami/kubectl"` |  |
| setupJob.machinekeyWriter.image.tag | string | `""` |  |
| setupJob.machinekeyWriter.resources | object | `{}` |  |
| setupJob.podAdditionalLabels | object | `{}` |  |
| setupJob.podAnnotations | object | `{}` |  |
| setupJob.resources | object | `{}` |  |
| startupProbe.enabled | bool | `true` |  |
| startupProbe.failureThreshold | int | `30` |  |
| startupProbe.periodSeconds | int | `1` |  |
| tolerations | list | `[]` |  |
| topologySpreadConstraints | list | `[]` |  |
| zitadel.configSecretKey | string | `"config-yaml"` |  |
| zitadel.configSecretName | string | `nil` |  |
| zitadel.configmapConfig.ExternalSecure | bool | `true` |  |
| zitadel.configmapConfig.FirstInstance.Org.LoginClient."Pat.ExpirationDate" | string | `"2029-01-01T00:00:00Z"` |  |
| zitadel.configmapConfig.FirstInstance.Org.LoginClient.Machine.Name | string | `"Automatically Initialized IAM Login Client"` |  |
| zitadel.configmapConfig.FirstInstance.Org.LoginClient.Machine.Username | string | `"login-client"` |  |
| zitadel.configmapConfig.Machine.Identification.Hostname.Enabled | bool | `true` |  |
| zitadel.configmapConfig.Machine.Identification.Webhook.Enabled | bool | `false` |  |
| zitadel.dbSslAdminCrtSecret | string | `""` |  |
| zitadel.dbSslCaCrt | string | `""` |  |
| zitadel.dbSslCaCrtAnnotations."helm.sh/hook" | string | `"pre-install,pre-upgrade"` |  |
| zitadel.dbSslCaCrtAnnotations."helm.sh/hook-delete-policy" | string | `"before-hook-creation"` |  |
| zitadel.dbSslCaCrtAnnotations."helm.sh/hook-weight" | string | `"0"` |  |
| zitadel.dbSslCaCrtSecret | string | `""` |  |
| zitadel.dbSslUserCrtSecret | string | `""` |  |
| zitadel.debug.annotations."helm.sh/hook" | string | `"pre-install,pre-upgrade"` |  |
| zitadel.debug.annotations."helm.sh/hook-weight" | string | `"1"` |  |
| zitadel.debug.enabled | bool | `false` |  |
| zitadel.debug.extraContainers | list | `[]` |  |
| zitadel.debug.initContainers | list | `[]` |  |
| zitadel.extraContainers | list | `[]` |  |
| zitadel.initContainers | list | `[]` |  |
| zitadel.masterkey | string | `""` |  |
| zitadel.masterkeyAnnotations."helm.sh/hook" | string | `"pre-install,pre-upgrade"` |  |
| zitadel.masterkeyAnnotations."helm.sh/hook-delete-policy" | string | `"before-hook-creation"` |  |
| zitadel.masterkeyAnnotations."helm.sh/hook-weight" | string | `"0"` |  |
| zitadel.masterkeySecretName | string | `""` |  |
| zitadel.revisionHistoryLimit | int | `10` |  |
| zitadel.secretConfig | string | `nil` |  |
| zitadel.secretConfigAnnotations."helm.sh/hook" | string | `"pre-install,pre-upgrade"` |  |
| zitadel.secretConfigAnnotations."helm.sh/hook-delete-policy" | string | `"before-hook-creation"` |  |
| zitadel.secretConfigAnnotations."helm.sh/hook-weight" | string | `"0"` |  |
| zitadel.selfSignedCert.additionalDnsName | string | `nil` |  |
| zitadel.selfSignedCert.enabled | bool | `false` |  |
| zitadel.serverSslCrtSecret | string | `""` |  |
<!-- end.chart.valuesTable -->

## Troubleshooting

### Debug Pod

For troubleshooting, you can deploy a debug pod by setting the `zitadel.debug.enabled` property to `true`.
You can then use this pod to inspect the Zitadel configuration and run zitadel commands using the zitadel binary.
For more information, print the debug pods logs using something like the following command:

```bash
kubectl logs rs/my-zitadel-debug
```

### migration already started, will check again in 5 seconds

If you see this error message in the logs of the setup job, you need to reset the last migration step once you resolved the issue.
To do so, start a [debug pod](#debug-pod) and run something like the following command:

```bash
kubectl exec -it my-zitadel-debug -- zitadel setup cleanup --config /config/zitadel-config-yaml
```

### Multiple Releases in Single Namespace

Read the comment for the value login.loginClientSecretPrefix

## Contributing

Lint the chart:

```bash
docker run -it --network host --workdir=/data --rm --volume $(pwd):/data quay.io/helmpack/chart-testing:v3.5.0 ct lint --charts charts/zitadel --target-branch main
```

Test the chart:

```bash
# Create KinD cluster
kind create cluster --config ./charts/zitadel/acceptance_test/kindConfig.yaml

# Test the chart
go test ./...
```

Watch the Kubernetes pods if you want to see progress.

```bash
kubectl get pods --all-namespaces --watch

# Or if you have the watch binary installed
watch -n .1 "kubectl get pods --all-namespaces"
```

## Contributors

<a href="https://github.com/zitadel/zitadel-charts/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=zitadel/zitadel-charts" />
</a>
