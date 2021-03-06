---
title: 控制 Ingress 流量
description: 介绍在服务网格 Istio 中如何配置外部公开服务。
weight: 30
keywords: [流量管理,ingress]
---

> 注意：此任务使用新的 [v1alpha3 流量管理 API](/zh/blog/2018/v1alpha3-routing/)。旧的 API 已被弃用，将在下一个 Istio 版本中删除。如果您需要使用旧版本，请按照[此处](https://archive.istio.io/v0.7/docs/tasks/traffic-management/)的文档操作。

在 Kubernetes 环境中，[Kubernetes Ingress 资源](https://kubernetes.io/docs/concepts/services-networking/ingress/) 用于指定应在群集外部公开的服务。在 Istio 服务网格中，更好的方法（也适用于 Kubernetes 和其他环境）是使用不同的配置模型，即 [Istio Gateway](/docs/reference/config/istio.networking.v1alpha3/#Gateway) 。 `Gateway` 允许将 Istio 功能（例如，监控和路由规则）应用于进入群集的流量。

此任务描述如何配置 Istio 以使用 Istio 在服务网格外部公开服务 `Gateway`。

## 前提条件

* 按照[安装指南中](/zh/docs/setup/)的说明设置 Istio 。

* 确保您当前的目录是 `istio` 目录。

* 启动 [httpbin]({{< github_tree >}}/samples/httpbin) 样本，该样本将用作要在外部公开的目标服务。

  如果您已启用[自动注入 sidecar](/zh/docs/setup/kubernetes/sidecar-injection/#sidecar-的自动注入)，请执行

{{< text bash >}}
$ kubectl apply -f @samples/httpbin/httpbin.yaml@
{{< /text >}}

  否则，您必须在部署 `httpbin` 应用程序之前手动注入边车：

{{< text bash >}}
$ kubectl apply -f <(istioctl kube-inject -f @samples/httpbin/httpbin.yaml@)
{{< /text >}}

* 按照以下小节中的说明确定 Ingress IP 和端口。

### 确定入口 IP 和端口

执行以下命令以确定您的 Kubernetes 群集是否在支持外部负载均衡器的环境中运行。

{{< text bash >}}
$ kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   172.21.109.129   130.211.10.121  80:31380/TCP,443:31390/TCP,31400:31400/TCP   17h
{{< /text >}}

如果 `EXTERNAL-IP` 设置了该值，则要求您的环境具有可用于 ingress 网关的外部负载均衡器。如果 `EXTERNAL-IP` 值是 `<none>`（或一直是 `<pending>` ），则说明可能您的环境不支持为 ingress 网关提供外部负载均衡器的功能。在这种情况下，您可以使用 Service 的 [node port](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) 方式访问网关。

#### 使用外部负载均衡器时确定 IP 和端口

{{< text bash >}}
$ export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http")].port}')
$ export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
{{< /text >}}

#### 确定使用 Node Port 时的 ingress IP 和端口

确定端口：

{{< text bash >}}
$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
$ export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
{{< /text >}}

确定 ingress IP  的具体方法取决于集群提供商。

1.  _GKE：_

    {{< text bash >}}
    $ export INGRESS_HOST=<workerNodeAddress>
    {{< /text >}}

    您需要创建防火墙规则以允许 TCP 流量进入 _ingress gateway_ 服务的端口。运行以下命令以允许 HTTP 端口，安全端口（HTTPS）或两者的流量。

    {{< text bash >}}
    $ gcloud compute firewall-rules create allow-gateway-http --allow tcp:$INGRESS_PORT
    $ gcloud compute firewall-rules create allow-gateway-https --allow tcp:$SECURE_INGRESS_PORT
    {{< /text >}}

1.  _IBM Cloud Kubernetes 服务免费版本：_

    {{< text bash >}}
    $ bx cs workers <cluster-name or id>
    $ export INGRESS_HOST=<public IP of one of the worker nodes>
    {{< /text >}}

1.  _Minikube：_

    {{< text bash >}}
    $ export INGRESS_HOST=$(minikube ip)
    {{< /text >}}

1.  _其他环境（例如IBM Cloud Private等）：_

    {{< text bash >}}
    $ export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o 'jsonpath={.items[0].status.hostIP}')
    {{< /text >}}

## 使用 Istio 网关配置 Ingress

Ingress [网关](/docs/reference/config/istio.networking.v1alpha3/#Gateway)描述了在网格边缘操作的负载平衡器，用于接收传入的 HTTP/TCP 连接。它配置暴露的端口，协议等，但与 [Kubernetes Ingress Resources](https://kubernetes.io/docs/concepts/services-networking/ingress/) 不同，它不包括任何流量路由配置。流入流量的流量路由使用 Istio 路由规则进行配置，与内部服务请求完全相同。

让我们看看如何为 `Gateway` 在 HTTP 80 端口上配置流量。

1.  创建一个 Istio `Gateway`

    {{< text bash >}}
    $ cat <<EOF | istioctl create -f -
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: httpbin-gateway
    spec:
      selector:
        istio: ingressgateway # use Istio default gateway implementation
      servers:
      - port:
          number: 80
          name: http
          protocol: HTTP
        hosts:
        - "httpbin.example.com"
    EOF
    {{< /text >}}

1.  为通过 `Gateway` 进入的流量配置路由

    {{< text bash >}}
    $ cat <<EOF | istioctl create -f -
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: httpbin
    spec:
      hosts:
      - "httpbin.example.com"
        gateways:
      - httpbin-gateway
        http:
      - match:
        - uri:
            prefix: /status
        - uri:
            prefix: /delay
        route:
        - destination:
            port:
              number: 8000
            host: httpbin
    EOF
    {{< /text >}}

    在这里，我们 为服务创建了一个[虚拟服务](/docs/reference/config/istio.networking.v1alpha3/#VirtualService)配置 `httpbin` ，其中包含两条路由规则，允许路径 `/status` 和 路径的流量 `/delay`。

    该[网关](/docs/reference/config/istio.networking.v1alpha3/#VirtualService-gateways)列表指定，只有通过我们的要求 `httpbin-gateway` 是允许的。所有其他外部请求将被拒绝，并返回 404 响应。

    请注意，在此配置中，来自网格中其他服务的内部请求不受这些规则约束，而是简单地默认为循环路由。要将这些（或其他规则）应用于内部调用，我们可以将特殊值 `mesh` 添加到 `gateways` 的列表中。

1.  使用 _curl_ 访问 _httpbin_ 服务。

    {{< text bash >}}
    $ curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/status/200
    HTTP/1.1 200 OK
    server: envoy
    date: Mon, 29 Jan 2018 04:45:49 GMT
    content-type: text/html; charset=utf-8
    access-control-allow-origin: *
    access-control-allow-credentials: true
    content-length: 0
    x-envoy-upstream-service-time: 48
    {{< /text >}}

    请注意，我们使用该 `-H` 标志将 _Host_  HTTP Header 设置为 "httpbin.example.com”。这是必需的，因为我们的 ingress `Gateway` 被配置为处理 "httpbin.example.com”，但在我们的测试环境中，我们没有该主机的 DNS 绑定，并且只是将我们的请求发送到 ingress IP。

1.  访问任何未明确公开的其他 URL。您应该看到一个 HTTP 404 错误：

    {{< text bash >}}
    $ curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/headers
    HTTP/1.1 404 Not Found
    date: Mon, 29 Jan 2018 04:45:49 GMT
    server: envoy
    content-length: 0
    {{< /text >}}

## 使用浏览器访问 Ingress 服务

正如您可能已经猜到的那样，在浏览器中输入 httpbin 服务 URL 是行不通的，因为我们没有办法告诉浏览器假装访问 `httpbin.example.com`，就像我们使用 _curl_ 一样。在现实世界中，这不会成为问题，因为所请求的主机将被正确配置并且 DNS 可解析，因此我们只需在 URL 中使用其域名（例如，`https://httpbin.example.com/status/200`）。

要解决此问题以进行简单的测试和演示，我们可以在 `Gateway` 和 `VirutualService` 配置中为主机使用通配符值 `*`。例如，如果我们将 ingress 配置更改为以下内容：

{{< text bash >}}
$ cat <<EOF | istioctl replace -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "*"
  gateways:
  - httpbin-gateway
    http:
  - match:
    - uri:
        prefix: /headers
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
{{< /text >}}

然后 `$INGRESS_HOST:$INGRESS_PORT`，我们可以在我们在浏览器中输入的 URL 中使用（例如：`192.168.99.100:31380`）。例如，`http://192.168.99.100:31380/headers` 应显示我们的浏览器发送的请求 Headers。

## 了解发生了什么

`Gateway` 配置资源允许外部流量进入 Istio 服务网，并使 Istio 的流量管理和策略功能可用于边缘服务。

在前面的步骤中，我们在 Istio 服务网格中创建了一个服务，并展示了如何将服务的 HTTP 端点暴露给外部流量。

## 清理

删除 `Gateway` 配置，`VirtualService` 密码并关闭 [httpbin]({{< github_tree >}}/samples/httpbin) 服务：

{{< text bash >}}
$ istioctl delete gateway httpbin-gateway
$ istioctl delete virtualservice httpbin
$ kubectl delete --ignore-not-found=true -f @samples/httpbin/httpbin.yaml@
{{< /text >}}
