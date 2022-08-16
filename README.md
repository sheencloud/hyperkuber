# HyperKuber

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) ![Release](https://github.com/sheencloud/hyperkuber/workflows/Release%20Manifests/badge.svg?branch=main) [![Releases downloads](https://img.shields.io/github/downloads/sheencloud/hyperkuber/total.svg)](https://github.com/sheencloud/hyperkuber/releases)

+ ## HyperKuber Container Management Platform (HKCMP)

The HyperKuber container management platform is a powerful tool for those who use kubernetes/containers as their daily job. It provides a centralized management portal for multiple clusters and removes the barriers to deploying applications across multiple clusters. As organizations' adoption of Kubernetes/containers grows, how to control, debug, and troubleshoot containerized applications becomes a pain for developers and engineers. In a multi-cluster environment, HyperKuber provides an effective and easy-to-manage solution for team collaboration and permission distribution. HyperKuber can not only help individuals quickly master the way kubernetes manages applications and improve productivity, but for enterprise organizations, HyperKuber can also serve as a platform for enterprise portals to provide containerized services to their customers.



+ ## HyperShift Container Management Platform (HSCMP)


Based on the HyperKuber container management platform, HyperShift adds support for OpenShift clusters and is compatible with the latest version of the OpenShift platform. Added management of OpenShift cluster resources, such as DeploymentConfig, Route, BuildConfig and other types. HyperShift can manage not only Openshift clusters, but also Kubernetes clusters. For OpenShift's build capabilities, HyperShift provides the functionality of building images from source code, binary, s2i, etc. With the help of OpenShift routing, HyperShift provides grayscale depoyment of applications.


## Installation

For quick start and demo try out, run the following command as a cluster admin:

```console
kubectl apply -f https://manifests.sheencloud.com/manifests/manifests.yaml
```

Note: Do not store sensitive data on this installation, when pod restarts, all data will be lost. For production installation, run the following command as a cluster admin:

```console
kubectl apply -f https://manifests.sheencloud.com/manifests/manifests-persistent.yaml
```

Check out pod status in namespace hyperkuber

```console
kubectl get po -n hyperkuber
```

When all pods are running and ready, access the console ingress created by default,

```console
kubectl get ing -n hyperkuber
```

Open the ingress in your favourite browser and login with default user/password: admin/hyperkuber@1314

For other installation methods, such as [helm installation](https://charts.sheencloud.com) or [operator installation](https://operator.sheencloud.com), checkout our [online documents](https://docs.sheencloud.com) for more details.


## Contributing

The source code of all [HyperKuber](https://sheencloud.com) community [Manifests](https://manifests.sheencloud.com)  can be found on Github: <https://github.com/sheencloud/hyperkuber/>

<!-- Keep full URL links to repo files because this README syncs from main to gh-pages.  -->
We'd love to have you contribute! Please refer to our [contribution guidelines](https://github.com/sheencloud/hyperkuber/blob/main/CONTRIBUTING.md) for details.

## License

<!-- Keep full URL links to repo files because this README syncs from main to gh-pages.  -->
[Apache 2.0 License](https://github.com/sheencloud/hyperkuber/blob/main/LICENSE).

## Release build status

![Release](https://github.com/sheencloud/hyperkuber/workflows/Release%20Manifests/badge.svg?branch=main)

# Resource

[Online Document](https://docs.sheencloud.com/home)

[Customer Center](https://account.sheencloud.com/)

[Discussion Forum](https://github.com/orgs/sheencloud/discussions)

[Slack Channel](https://sheencloud-workspace.slack.com)
