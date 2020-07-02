gRPC Retry Design
----
* Author(s): [Noah Eisen](https://github.com/ncteisen) and [Eric Gribkoff](https://github.com/ericgribkoff)
* Approver: a11r
* Status: Ready for Implementation
* Implementation: In progress for all language stacks
* Last updated: 2017-09-13
* Discussion at: https://groups.google.com/forum/#!topic/grpc-io/zzHIICbwTZE

Table of Contents
----
  * [Abstract](#abstract)
  * [Background](#background)
  * [Proposal](#proposal)
     * [Overview](#overview)
     * [Detailed Design](#detailed-design)
        * [Retry Policy Capabilities](#retry-policy-capabilities)
           * [Maximum Number of Retries](#maximum-number-of-retries)
           * [Exponential Backoff](#exponential-backoff)
           * [Retryable Status Codes](#retryable-status-codes)
        * [Hedging Policy](#hedging-policy)
        * [Throttling Retry Attempts and Hedged RPCs](#throttling-retry-attempts-and-hedged-rpcs)
        * [Pushback](#pushback)
        * [Limits on Retries and Hedges](#limits-on-retries-and-hedges)
        * [Summary of Retry and Hedging Logic](#summary-of-retry-and-hedging-logic)
     * [Retry Internals](#retry-internals)
        * [Where Retries Occur](#where-retries-occur)
        * [When Retries are Valid](#when-retries-are-valid)
        * [Memory Management (Buffering)](#memory-management-buffering)
        * [Transparent Retries](#transparent-retries)
        * [Exposed Retry Metadata](#exposed-retry-metadata)
        * [Disabling Retries](#disabling-retries)
        * [Retry and Hedging Statistics](#retry-and-hedging-statistics)
     * [Configuration Language](#configuration-language)
        * [Retry Policy](#retry-policy)
        * [Hedging Policy](#hedging-policy-1)
        * [Throttling Configuration](#throttling-configuration)
        * [Integration with Service Config](#integration-with-service-config)

## Abstract

gRPC client library will automatically retry failed RPCs according to a policy set by the service owner.

gRPC 客户端库可以根据服务设置的策略对失败的请求自动重试

## Background 背景

Currently, gRPC does not retry failed RPCs. All failed RPCs are immediately returned to the application layer by the gRPC client library.

Many teams have implemented their own retry logic wrapped around gRPC like [Veneer Toolkit](https://github.com/googleapis/toolkit) and [Cloud Bigtable](https://github.com/GoogleCloudPlatform/cloud-bigtable-client).

当前 gRPC没有重试，所有失败的 RPC 请求被立即返回给应用层，有很多团队通过封装实现了自己的重试策略，如 [Veneer Toolkit](https://github.com/googleapis/toolkit) 和 [Cloud Bigtable](https://github.com/GoogleCloudPlatform/cloud-bigtable-client)

## Proposal

### Overview

gRPC will support two configurable retry policies. The [service configuration](https://github.com/grpc/grpc/blob/master/doc/service_config.md) (which will soon be [published via DNS](https://github.com/grpc/proposal/pull/5)) may choose from a retry policy (retry failed RPCs) or a hedging policy (aggressively send the same RPC multiple times in parallel). An individual RPC may be governed by a retry policy or a hedge policy, but not both.

gRPC 支持配置两种重试策略，[服务配置](https://github.com/grpc/grpc/blob/master/doc/service_config.md)支持选择重试失败的请求，或者对冲(同时发出多个请求)，一个单独的 RPC 可以使用重试策略，或者对冲策略，但是不能同时使用

Retry policy capabilities are as follows. Each has a detailed description below.
重试策略的能力如下：
* [最大重试次数](#maximum-number-of-retries)
* [指数退避](#exponential-backoff)
* [重试状态码](#retryable-status-codes)

The hedging policy has the following parameters. See details [here](#hedging-policy).
对冲策略的参数如下，可以参考 [对冲策略](#hedging-policy).
* 最大对冲请求数
* 对冲请求时间间隔
* 非失败状态码

Additionally, gRPC provides a mechanism to throttle retry attempts and hedged RPCs when the ratio of failures to successes exceeds a threshold. See [detailed description of throttling](#throttling-retry-attempts-and-hedged-rpcs).
gRPC 提供了机制，当失败与成功的比率超过阈值时，将停止重试和对冲，可以参考 [节流限制](#throttling-retry-attempts-and-hedged-rpcs).

We also provide a mechanism for servers to explicitly signal clients to retry after a settable delay. See [detailed description of server pushback](#pushback).
还提供了一种机制，服务端可以发出显式的信号通知客户端在指定延迟后开始重试，参考 [服务推送的详细描述](#pushback)


In some cases, gRPC can guarantee that a request has never been seen by the server application logic. These cases will be transparently retried by gRPC, as detailed [here](#transparent-retries).
在一些场景下，gRPC可以保证请求不会到达服务端的逻辑处理，这些场景应该由 gRPC 透明的进行重试，详细可以参考 [透明重试](#transparent-retries)

Lastly, information about number of retry attempts will be exposed to the client and server applications through metadata. Find more details [here](#exposed-retry-metadata).
最后，有关重试的次数会通过元数据暴露给客户端和服务端，参考[暴露重试元数据](#exposed-retry-metadata)

![State Diagram](A6_graphics/basic_retry.png)

[Link to SVG file](A6_graphics/basic_retry.svg)

### Detailed Design 详细设计

#### Retry Policy Capabilities 重试策略

Retry policies support configuring the maximum number of retries, the parameters for exponential backoff, and the set of retryable status codes, as follows:
重试策略支持配置最大重试次数，指数退避的参数，一组可以重试的状态，如：

```
"retryPolicy": {
  "maxAttempts": 4,
  "initialBackoff": "0.1s",
  "maxBackoff": "1s",
  "backoffMultiplier": 2,
  "retryableStatusCodes": [
    "UNAVAILABLE"
  ]
}
```
Each of these configuration options is detailed in its own section below.
每个配置都可以参考下面的详细设计.

##### Maximum Number of Retries

`maxAttempts` specifies the maximum number of RPC attempts, including the original request.
`maxAttempts` 指定 RPC 最大重试次数，包括原始请求

gRPC's call deadline applies across all attempts for a given RPC. For example, if the specified deadline for an RPC is July 23 9:00:00pm PDT the operation will fail after that time regardless of how many attempts were configured or attempted.
gPRC调用期限适用于给定 RPC 的所有重试；如，一个指定的RPC的重试期限是 7-23 21:00:00，过了这个时间，不论配置了多少次重试，都会失败

![State Diagram](A6_graphics/too_many_attempts.png)

[Link to SVG file](A6_graphics/too_many_attempts.svg)

##### Exponential Backoff 指数退避

The `initialBackoff`, `maxBackoff`, and `backoffMultiplier` parameters determine the randomized delay before retry attempts.
参数 `initialBackoff`, `maxBackoff`, 和 `backoffMultiplier` 随机决定下一次重试之前的延时

The initial retry attempt will occur at `random(0, initialBackoff)`. In general, the `n`-th attempt will occur at `random(0, min(initialBackoff*backoffMultiplier**(n-1), maxBackoff))`.
初始的重试会发生在 `random(0, initialBackoff)` 之间，随后的第n次重试会发生在 `random(0, min(initialBackoff*backoffMultiplier**(n-1), maxBackoff))`

##### Retryable Status Codes 重试状态

When gRPC receives a non-OK response status from a server, this status is checked against the set of retryable status codes in `retryableStatusCodes` to determine if a retry attempt should be made.
当 gRPC 返回的状态不是 OK 时，会根据 `retryableStatusCodes` 检查是否是重试状态，决定是否要重试

In general, only status codes that indicate the service did not process the request should be retried. However, a more aggressive set of parameters can be specified when the service owner knows that a method is idempotent, or safe to be processed more than once by the server. For example, if an RPC to a delete user method fails with an `INTERNAL` error code, it's possible that the user was already deleted before the failure occurred. If the method is idempotent then retrying the call should have no adverse effects. However, it is entirely possible to implement a non-idempotent delete method that has adverse side effects if called multiple times, and such a method should not be retried unless the error codes guarantee the original RPC was not processed. It is up to the service owner to pick the correct set of retryable status codes given the semantics of their service. gRPC retries do not provide a mechanism to specifically mark a method as idempotent.
通常情况下，只有表明服务没有处理请求的状态码才应该重试，但是当知道服务是幂等或者安全，可以支持多次请求的时候可以指定更多的状态；如一个删除用户的方法失败并返回 `INTERNAL` 错误，很有可能在失败之前用户已经被删除了；如果该方法是幂等的，则重试不会产生影响，然而，很可能是一个多次调用会产生不良后果的非幂等方法，除非错误码明确
告知原始的请求没有处理，取决于服务的所有者选择正确的可重试状态集，gRPC重试机制没有提供表明是否是幂等的机制


##### Validation of retryPolicy 验证重试策略

If `retryPolicy` is specified in a service config choice, the following validation rules apply:
1. `maxAttempts` MUST be specified and MUST be a JSON integer value greater than 1. Values greater than 5 are treated as 5 without being considered a validation error.
2. `initialBackoff` and `maxBackoff` MUST be specified, MUST follow the JSON representaion of [proto3 Duration type](https://developers.google.com/protocol-buffers/docs/proto3#json), and MUST have a duration value greater than 0.
3. `backoffMultiplier` MUST be specified and MUST be a JSON number greater than 0.
4. `retryableStatusCodes` MUST be specified as a JSON array of status codes and be non-empty. Each status code MUST be a valid gRPC status code and specified in the integer form or the case-insensitive string form (eg. [14], ["UNAVAILABLE"] or ["unavailable"]).
如果服务配置了 `retryPolicy`，以下规则将会生效
1. `maxAttempts` 必须是一个大于 1 的JSON数值，当大于5时将会被替换为5，不会报错
2. `initialBackoff` 和 `maxBackoff` 必须配置，必须遵循 [proto3 Duration type](https://developers.google.com/protocol-buffers/docs/proto3#json)的规则，且必须大于0
3. `backoffMultiplier` 必须配置，且必须是个大于0的 JSON 数值
4. `retryableStatusCodes` 必须是一个非空的 JSON 数组，每个状态码必须是有效的 gRPC 状态码，可以是数值或者大小写不敏感的状态码(如. [14], ["UNAVAILABLE"] or ["unavailable"]).

#### Hedging Policy 对冲策略

Hedging enables aggressively sending multiple copies of a single request without waiting for a response. Because hedged RPCs may be be executed multiple times on the server side, typically by different backends, it is important that hedging is only enabled for methods that are safe to execute multiple times without adverse affect.
开启对冲策略会侵入性的发送多个同样的请求而不必等待返回结果，因为对冲的请求可能会在不同的 Server 端执行多次，只有在可以安全执行多次且没有影响的方法上开启对冲


Hedged requests are configured with the following parameters:
对冲请求配置参数如下：

```
"hedgingPolicy": {
  "maxAttempts": 4,
  "hedgingDelay": "0.5s",
  "nonFatalStatusCodes": [
    "UNAVAILABLE",
    "INTERNAL",
    "ABORTED"
  ]
}
```

When a method has chosen a `hedgingPolicy`, the original RPC is sent immediately, as with a standard non-hedged call. After `hedgingDelay` has elapsed without a successful response, the second RPC will be issued. If neither RPC has received a response after `hedgingDelay` has elapsed again, a third RPC is sent, and so on, up to `maxAttempts`. In the above configuration, after 1ms there would be one outstanding RPC (the original), after 501ms there would be two outstanding RPCs (the original and the first hedged RPC), after 1001ms there would be three outstanding RPCs, and after 1501ms there would be four. As with retries, gRPC call deadlines apply to the entire chain of hedged requests. Once a deadline has passed, the operation fails regardless of in-flight RPCS, and regardless of the hedging configuration.
当方法启用了对冲策略后，原始的RPC请求以非对冲的方式会被立即发送，当 `hedgingDelay` 时间之后请求没有成功返回，会发出第二个请求，当再次过了 `hedgingDelay` 之后依然没有返回结果时，会发出第三个请求，持续如此，直到达到最大的重试次数 `maxAttempts`，在上面的配置中，1ms会有一个未完成的 RPC(原始的)，501ms后会有两个未完成的 RPC请求(原始的RPC请求和第一个对冲的 RPC)，1001ms后会有三个，1501ms后会有四个，与重试一样，RPC的调用期限适用于所有对冲的 RPC请求，一旦到达期限，不管运行的 RPC请求是什么，也不管对冲配置是什么，请求都会失败


The implementation will ensure that the listener returned to the client application forwards its calls (such as `onNext` or `onClose`) to all outstanding hedged RPCs.
重试的实现会确保监听器会将未完成的对冲请求通过 `onNext` 或 `onClose` 返回给客户端

When a non-error response is received (in response to any of the hedged requests), all outstanding hedged requests are canceled and the response is returned to the client application layer.
当接收到一个不是错误的响应时(任意一个对冲请求的)，取消其他所有对冲请求，并将相应返回给应用层

If a non-fatal status code is received from a hedged request, then the next hedged request in line is sent immediately, shortcutting its hedging delay. If any other status code is received, all outstanding RPCs are canceled and the error is returned to the client application layer.
如果收到的响应是非失败的状态码，那么马上会发出一个对冲请求，缩短对冲延迟时间；如果接收到其他的状态的响应，所有的对冲请求都会被取消，并将错误返回给应用层

If all instances of a hedged RPC fail, there are no additional retry attempts. Essentially, hedging can be seen as retrying the original RPC before a failure is even received.
如果所有实例的对冲请求都失败了，则不会再执行重试，从本质上讲，对冲也可以看做是收到请求失败之前的重试

If server pushback that specifies not to retry is received in response to a hedged request, no further hedged requests should be issued for the call.
如果收到指定不重试的服务器回推作为对已对冲请求的响应，则不应为对该调用再发起对冲请求

Hedged requests should be sent to distinct backends, if possible. To facilitate this, the gRPC client will maintain a list of previously used backend addresses for each hedged RPC. This list will be passed to the gRPC client's local load-balancing policy. The load balancing policy may use this information to send the hedged request to an address that was not previously used. If all available backend addresses have already been used, the load-balancing policy's response is implementation-dependent.
对冲请求应当发给不同的服务端，为了实现这个，gRPC客户端将为每个已对冲RPC维护之前使用的后端地址列表，这个列表会传递给 gRPC的客户端用于负载均衡策略，负载均衡会根据这些信息选择一个之前没有发送过对冲请求的服务器，如果所有的服务端实例都被发送过对冲请求，那么根据负载均衡的实现选择


##### Validation of hedgingPolicy 对冲策略校验

If `hedgingPolicy` is specified in a service config choice, the following validation rules apply:
1. `maxAttempts` MUST be specified and MUST be a JSON integer value greater than 1. Values greater than 5 are treated as 5 without being considered a validation error.
2. `hedgingDelay` is an optional field but if specified MUST follow the JSON representation of proto3 Duration type.
3. `nonFatalStatusCodes` is an optional field but if specified MUST be specified as a JSON array of status codes. Each status code MUST be a valid gRPC status code and specified in the integer form or the case-insensitive string form (eg. [14], ["UNAVAILABLE"] or ["unavailable"]).

如果对服务配置了 `hedgingPolicy`，那么以下配置就会生效：
1. `maxAttempts`必须指定并且必须是大于1的JSON 数值，大于5的值将不会生效，会替换为5
2. `hedgingDelay` 是一个可选的属性，但是指定时必须遵守 proto3 Duration JSON 的格式
3. `nonFatalStatusCodes` 是一个可选的属性，但是指定时必须是遵守 JSON 数组格式的状态码，每个状态码必须是有效的，可以是相应的数值或者大小写不敏感的字符(如 [14], ["UNAVAILABLE"] 或 ["unavailable"])

![State Diagram](A6_graphics/basic_hedge.png)

[Link to SVG file](A6_graphics/basic_hedge.svg)

#### Throttling Retry Attempts and Hedged RPCs 限制重试和对冲 RPC

gRPC prevents server overload due to retries and hedged RPCs by disabling these policies when the client’s ratio of failures to successes passes a certain threshold. The throttling is done per server name. Retry throttling may be configured as follows:
gRPC 为了避免重试或者对冲导致服务端超载，支持通过客户端的失败和成功的比率停止重试或者对冲，节流是按服务器名进行的，重试的节流配置如下：

```
"retryThrottling": {
  "maxTokens": 10,
  "tokenRatio": 0.1
}
```

Throttling may only be specified per server name, rather than per method or per service.
只能按服务器名指定节流，而不能根据方法或者服务名

For each server name, the gRPC client maintains a `token_count` variable which is initially set to `maxTokens` and can take values between `0` and `maxTokens`. Every outgoing RPC (regardless of service or method invoked) will effect `token_count` as follows:
* Every failed RPC will decrement the `token_count` by 1.
* Every successful RPC will increment the `token_count` by `tokenRatio`.
对于每一个服务名，gRPC 客户端记录了初始值为 `maxTokens`，范围在 `0` 和 `maxTokens` 之间的 `token_count` 变量，每个 RPC 请求都会影响`token_count`的值：
* 每个失败的请求会使 `token_count` 减一
* 每个成功的请求会使 `token_count` 增加 `tokenRatio`

If `token_count` is less than or equal to the threshold, defined to be `(maxTokens / 2)`, then RPCs will not be retried until `token_count` rises over the threshold.
如果 `token_count` 小于等于 `(maxTokens / 2)` 的临界值，则 RPC 将不会重试，直到 `token_count` 再次超过临界值

Throttling also applies to hedged RPCs. The first outgoing RPC will always be sent, but subsequent hedged RPCs will only be sent if `token_count` is greater than the threshold.
临界值同样适用于对冲请求，第一个请求一定会被发送，只有当 `token_count` 大于临界值的时候后续对冲请求才会发送

Neither retry attempts or hedged RPCs block when `token_count` is less than or equal to the threshold. Retry attempts are canceled and the failure returned to the client application. The hedged request is cancelled, and if there are no other already-sent hedged RPCs the failure is returned to the client application.
当 `token_count` 小于等于临界值时，无论是重试还是对冲请求都不会被发送，重试会取消并将失败返回给客户端，对冲将会被取消，如果没有已经发送的对冲请求那么会将失败返回给客户端

The only RPCs that are counted as failures for the throttling policy are RPCs that fail with a status code that qualifies as a [retryable](#retryable-status-codes) or [non-fatal status code](#hedging-policy), or that receive a pushback response indicating not to retry. This avoids conflating server failure with responses to malformed requests (such as the `INVALID_ARGUMENT` status code).
对于节流策略而言，只有失败的状态码或者指定需要重试的状态码，或者指定非失败的状态码，或者server端返回的指定不需要重试的才会被算入节流策略的失败数量中，这避免了服务器故障和与对格式错误的请求的响应混为一谈(如`INVALID_ARGUMENT`状态码)

##### Validation of retryThrottling 重试节流校验

If `retryThrottling` is specified in a service config, the following validation rules apply:
1. `maxTokens` MUST be specified and MUST be a JSON integer value in the range (0, 1000].
2. `tokenRatio` MUST be specified and MUST be a JSON floating point greater than 0. Decimal places beyond 3 are ignored. (eg. 0.5466 is treated as 0.546.)

如果服务指定了 `retryThrottling` ，以下校验就会生效：
1. `maxTokens`必须指定并且必须是 JSON 格式的的整数值，范围为 (0,1000]
2. `tokenRatio` 必须指定并且必须是大于0的 JSON 格式的浮点数，小数点后三位有效，超过的将会被忽略

#### Pushback 回推

Servers may explicitly pushback by setting metadata in their response to the client. The pushback can either tell the client to retry after a given delay or to not retry at all. If the client has already exhausted its `maxAttempts`, the call will not be retried even if the server says to retry after a given delay.
服务的可以通过在响应中返回元数据给客户端显式回推，回推可以告诉客户端是否在给定延时后重试或者完全不重试，如果客户端已经达到最大重试次数 `maxAttempts`，即使服务端返回在指定延时后重试，客户端也不会执行

Pushback may also be received to a hedged request. If the pushback says not to retry, no further hedged requests will be sent. If the pushback says to retry after a given delay, the next hedged request (if any) will be issued after the given delay has elapsed.
对冲的请求也可以接收回推，如果回推告诉客户端不要重试，那么不会有对冲请求继续发出，如果回推告诉客户端在指定延时后重试，下一个对冲请求(如果有的话)将会在给定的延迟后发出

A new metadata key, `"grpc-retry-pushback-ms"`, will be added to support server pushback. The value is to be an ASCII encoded signed 32-bit integer with no unnecessary leading zeros that represents how many milliseconds to wait before sending a retry. If the value for pushback is negative or unparseble, then it will be seen as the server asking the client not to retry at all.
将会添加一个新的元数据 `"grpc-retry-pushback-ms"`用于支持回推，值是一个 ASCII 编码的有符号的32位整数，表示在重试之前需要等待多少毫秒，如果值是一个负数或者不能解析，将被视为服务端要求不要重试

When a client receives an explicit pushback response from a server, and it is appropriate to retry the RPC, it will retry after exactly that delay.  For subsequent retries, the delay period will be reset to the `initialBackoff` setting and scale according to the [Exponential Backoff](#exponential-backoff) section above, unless an explicit pushback response is received again.
当客户端接收到服务端的明确的回推响应，可以重试，则会在指定的延迟后开始重试，后续的重试中，延时间隔将会被重置为 `initialBackoff`，并根据上面的 [指数退避](#exponential-backoff)部分缩放，除非再次收到回推响应

#### Limits on Retries and Hedges 重试和对冲限制

`maxAttempts` in both `retryPolicy` and `hedgingPolicy` have, by default, a client-side maximum value of 5. This client-side maximum value can be changed by the client through the use of channel arguments. Service owners may specify a higher value for these parameters, but higher values will be treated as equal to the maximum value by the client implementation. This mitigates security concerns related to the service config being transferred to the client via DNS.
`retryPolicy` 和 `hedgingPolicy` 都有 `maxAttempts` 参数，默认情况下，客户端的最大值为5，可以通过 channel 的参数修改，服务负责人可能会指定一个更高的值，但是更高的值将会客户端视为等于最大值，这减轻了与通过DNS传输到客户端的服务配置相关的安全问题


#### Summary of Retry and Hedging Logic 重试和对冲逻辑总结
There are five possible types of server responses. The list below enumerates the behavior of retry and hedging policies for each type of response. In all cases, if the maximum number of retry attempts or the maximum number of hedged requests is reached, no further RPCs are sent. Hedged RPCs are returned to the client application when all outstanding and pending requests have either received a response or been canceled.
服务端可能返回五种类型的响应，以下列举了每种响应的重试和对冲的策略；所有情况下，如果达到了最大重试次数或者最大对冲次数，则不会继续发送 RPC 请求，当所有的未完成和挂起的请求收到响应或者取消时，已对冲的 RPC 将返回给客户端应用

1. OK
    1. Retry policy: Successful response, return success to client application
    2. Hedging policy: Successful response, cancel previous and pending hedges

2. Fatal Status Code
    1. Retry policy: Don't retry, return failure to client application
    2. Hedging policy: Cancel previous and pending hedges

3. Retryable/Non-Fatal Status Code without Server Pushback
    1. Retry policy: Retry according to policy
    2. Hedging policy: Immediately send next scheduled hedged request, if any. Subsequent hedged requests will resume at `hedgingDelay`

4. Pushback: Don't Retry
    1. Retry policy: Don't retry, return failure to client application
    2. Hedging policy: Don’t send any more hedged requests.

5. Pushback: Retry in *n* ms
    1. Retry policy: Retry in *n* ms. If this attempt also fails, retry delay will reset to initial backoff for the following retry (if applicable)
    2. Hedging policy: Send next hedged request in *n* ms. Subsequent hedged requests will resume at `n + hedgingDelay`

1. OK 
    1. 重试策略：成功响应，返回成功给客户端应用
    2. 对冲策略：成功响应，取消之前和在等待的对冲请求
2. 失败状态码
    1. 重试策略：不会重试，返回失败给客户端应用
    2. 对冲策略：取消之前和在等待的对冲请求
3. 重试、非失败的没有回推的状态码
    1. 重试策略：根据策略进行重试
    2. 对冲策略：如果有则立即发送下一个计划中的对冲请求，随后的对冲请求将会在 `hedgingDelay` 后发出
4. 回推：不重试
    1. 重试策略：不重试，返回失败给客户端应用
    2. 对冲策略：不发送任何对冲请求
5. 回推：*n*ms 后重试
    1. 重试策略：在 *n* ms 后重试，如果这次重试也失败了，且有后续重试请求，下一个重试请求的延时将被重置为初始延迟时间         
    2. 对冲策略： *n* ms 后发出下一个对冲请求，随后的对冲请求将会在 `n + hedgingDelay` 后继续

![State Diagram](A6_graphics/StateDiagram.png)

[Link to SVG file](A6_graphics/StateDiagram.svg)

### Retry Internals 内部重试

#### Where Retries Occur 重试在哪里发生

The retry policy will be implemented in-between the channel and the load balancing policy. That way every retry gets a chance to be sent out on a different subchannel than it originally failed on.
重试会在 channel 和负载均衡策略之间实现，当原始请求失败之后有机会将请求转发给其他的 subchannel

![Where Retries Occur](A6_graphics/WhereRetriesOccur.png)

[Link to SVG file](A6_graphics/WhereRetriesOccur.svg)

#### When Retries are Valid 什么时候重试是有效的

In certain cases it is not valid to retry an RPC. These cases occur when the RPC has been *committed*, and thus it does not make sense to perform the retry.
有些情况下，重试 RPC 是无效的，这些情况发生在 RPC 已经被执行的时候，因此这些重试没有意义

An RPC becomes *committed* in two scenarios:

1. The client receives [Response-Headers](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md#responses).
2. The client’s outgoing message has overflowed the gRPC client library’s buffer.

RPC 在以下两种情况下会被执行：
1. 客户端收到 [Response-Headers](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md#responses).
2. 客户端发出的信息溢出了 gRPC 客户端的缓冲区

The reasoning behind the first scenario is that the Response-Headers include initial metadata from the server. The metadata (or its absence) it is transmitted to the client application. This may fundamentally change the state of the client, so we cannot safely retry if a failure occurs later in the RPC’s life.
第一个场景背后的原因是响应头包含服务端返回的初始元数据，元数据传输给客户端，可能会从根本上改变客户端的状态，因此，如果在 RPC 生命周期后期发生故障，则不能安全的进行重试

gRPC servers should delay the Response-Headers until the first response message or until the application code chooses to send headers. If the application code closes the stream with an error before sending headers or any response messages, gRPC servers should send the error in Trailers-Only.
gRPC 服务端应当延迟响应头，知道第一个响应消息或者状态码被发送到响应头中，如果应用状态因为错误在发送响应头或者响应消息之前关闭了流，gRPC 服务端应当发送Trailers-Only 的错误

To clarify the second scenario, we define an *outgoing message* as everything the client sends on its connection to the server. For unary and server streaming calls, the outgoing message is a single message. For client and bidirectional streaming calls, the outgoing message is the entire message stream issued by the client after opening the connection. The gRPC client library buffers outgoing messages, and as long as the entirety of the outgoing message is in the buffer, it can be resent and retried. But as soon as the outgoing message grows too large to buffer, the gRPC client library cannot replay the entire stream of messages, and thus retries are not valid.
为了明确第二个场景，将 *outgoing message* 定义为客户端在其连接到服务器上发送的所有消息，对于 unary 和 sever stream 调用，发送的下消息是单独的消息，对于双向的流调用，传出消息是客户端在打开连接后发出的整个消息流，gRPC客户端库缓冲传出的消息，只要传出的消息的整个都在缓冲区中，就可以重新发送和重试；但是一旦传出的消息变得太大而无法缓冲，gRPC客户端库就不能重播整个消息流，因此重试无效

#### Memory Management (Buffering) 内存管理(缓冲)

The gRPC client library will support application-configured limits for the amount of memory used for retries. It is suggested that the client sets `retry_buffer_size_in_bytes` to limit the total amount of memory used to buffer retryable or hedged RPCs. The client should also set `per_rpc_buffer_limit_in_bytes` to limit the amount of memory used by any one RPC (to prevent a single large RPC from using the whole buffer and preventing retries of subsequent smaller RPCs). These limits are configured by the client, rather than coming from the service config.
gRPC 客户端库支持配置用于重试的内存限制，建议客户端配置 `retry_buffer_size_in_bytes` 用于限制重试或对冲的缓存，同时客户端应当配置 `per_rpc_buffer_limit_in_bytes` 用于限制每一个 RPC 的缓冲，防止单个大型RPC使用整个缓冲区，并防止后续较小RPC的重试，这些限制配置在客户端，而不是服务端

RPCs may only be retried when they are contained in the buffer. New RPCs which do not fit in the available buffer space (either due to the total available buffer space, or due to the per-RPC limit) will not be retryable, but the original RPC will still be sent.
只有当 RPC 在缓冲中时才可以重试，新的重试 RPC 如果不适合当前的缓冲，不会重试，但是原始的请求依然会发送

After the RPC response has been returned to the client application layer, the RPC is removed from the buffer.
当 RPC 响应返回给客户端时，RPC 将会从缓冲中移除

Client streaming RPCs will take up additional buffer space with each subsequent message, so additional buffering policy is needed. When the application sends a message on an RPC that causes the RPC to exceed the buffer limit, the RPC becomes committed, meaning that we choose one attempt to continue and stop all others.
客户端的 stream 请求将为每个后续消息占用额外的缓冲区，所以额外缓冲区策略是必须的，如果应用发送了一个消息超过了缓冲区限制，这个 RPC 将会被提交，意味着只选择一个进行尝试并阻止其他尝试

For retriable RPCs, when an RPC becomes committed, the client will continue with the currently in-flight attempt but will not make any subsequent attempts, even if the current attempt fails with a retryable status and there are retry attempts remaining.
对于可重试的 RPC，当 RPC 提交后，客户端将会继续当前的重试，但是其他的重试不会继续，即使当前的重试失败且有重试尝试

For hedged RPCs, when an RPC becomes committed, the client will continue the currently in-flight attempt on which the maximum number of messages have already been sent. All other currently in-flight attempts will be immediately cancelled, and no subsequent attempts will be started.
对于对冲的 RPC，当 RPC 提交后，客户端会继续已经发送的数量最大的请求，其他的所有执行中的会立即取消，后续的请求不会执行

Once we have committed to a particular attempt, any messages that have already been sent on the that attempt can be immediately freed from the buffer, and each subsequent message that is replayed can be freed as soon as it is sent. The new message sent by the application (which caused the RPC to exceed the buffer limit) will never be added to the buffer in the first place; it will stay pending until all previous messages have been replayed, and then it will be sent immediately without buffering.
一旦提交了特定的尝试，尝试中的其他已经发送的请求会立即释放缓冲区，后续的请求在发送后会立即释放缓冲区；应用的新的请求如果超过缓冲区限制，则不会加到缓冲区；会持续等待直到所有值钱的消息都已经发送，然后会不经过缓冲区立即发送

When an RPC is evicted from the buffer, pending hedged requests should be canceled immediately. Implementations must avoid potential scheduling race conditions when canceling pending requests concurrently with incoming responses to already sent hedges, and ensure that failures are relayed to the client application logic when no more hedged requests will be possible and all outstanding requests have returned.
当 RPC 从缓冲区移除之后，等待对冲的请求应当被立即取消，实现必须避免潜在的调度竞争条件时取消挂起的请求与传入的响应已发送的限制，并确保在不再有任何被对冲的请求并且所有未完成的请求都已返回时，将故障转发到客户端应用程序逻辑

#### Transparent Retries 透明重试

RPC failures can occur in three distinct ways:

1. The RPC never leaves the client.
2. The RPC reaches the server, but has never been seen by the server application logic.
3. The RPC is seen by the server application logic, and fails.

RPC 有三种失败方式：
1. RPC 没有离开客户端
2. RPC 到达了服务端，但是没有发送给服务端的逻辑处理
3. RPC 经过服务端的逻辑处理，但是失败了

![Where RPCs Fail](A6_graphics/WhereRPCsFail.png)

[Link to SVG file](A6_graphics/WhereRPCsFail.svg)

The last case is handled by the configurable retry policy that is the main focus of this document. The first two cases are retried automatically by the gRPC client library, **regardless** of the retry configuration set by the service owner. We are able to do this because these request have not made it to the server application logic, and thus are always safe to retry.
最后一种失败场景是配置的重试策略处理的，也是这个文档的关注点，前两种场景会通过 gRPC 客户端库自动重试，重试配置的 **regardless** 由服务所有者自己配置；之所以有客户端自动重试是因为请求没有到达服务端处理逻辑，所以是可以安全重试的

In the first case, in which the RPC never leaves the client, the client library can transparently retry until a success occurs, or the RPC's deadline passes.
在第一种情况下，请求没有离开客户端，客户端可以透明重试，直到成功，所以 RPC 到达超时时间

If the RPC reaches the gRPC server library, but has never been seen by the server application logic (the second case), the client library will immediately retry it once. If this fails, then the RPC will be handled by the configured retry policy. This extra caution is needed because this case involves extra load on the wire.
如果 RPC 到达了服务端库，但是没有发送给应用逻辑处理(第二种场景)，客户端会立即重试，如果失败了，会根据重试策略进行处理，需要格外小心，因为这种情况涉及到线路上的额外负载

Since retry throttling is designed to prevent server application overload, and these transparent retries do not make it to the server application layer, they do not count as failures when deciding whether to throttle retry attempts.
由于重试调节是为了防止服务器应用程序过载而设计的，而且这些透明的重试不会到达服务器应用程序层，因此在决定是否限制重试尝试时，它们不会被视为失败

Similarly, transparent retries do not count toward the limit of configured RPC attempts (`maxAttempts`).
同样的，透明重试也不会算在 RPC 的重试次数 `maxAttempts` 中

![State Diagram](A6_graphics/transparent.png)

[Link to SVG file](A6_graphics/transparent.svg)

#### Exposed Retry Metadata 暴露重试元数据

Both client and server application logic will have access to data about retries via gRPC metadata. Upon seeing an RPC from the client, the server will know if it was a retry, and moreover, it will know the number of previously made attempts. Likewise, the client will receive the number of retry attempts made when receiving the results of an RPC.
客户端和服务端的应用逻辑都可以获取到重试的 gRPC 元数据，在看到来自客户端的请求时，服务端可以知道是不是重试，而且还可以知道之前重试的次数，同样，在收到 RPC 的响应时，可以知道重试的次数

The header name for exposing the metadata will be `"grpc-previous-rpc-attempts"` to give clients and servers access to the attempt count. This value represents the number of preceding retry attempts. Thus, it will not be present on the first RPC, will be 1 for the second RPC, and so on. The value for this field will be an integer.
将会在 header 里面暴露元数据 `"grpc-previous-rpc-attempts"` 用于客户端和服务端获取重试次数，此值代表之前尝试的次数，因此，在第一次请求时没有该值，第二次请求时是1，以此类推，这个属性的值是整数值

#### Disabling Retries 禁止重试

Clients cannot override retry policy set by the service config. However, retry support can be disabled entirely within the gRPC client library. This is designed to enable existing libraries that wrap gRPC with their own retry implementation (such as Veneer Toolkit) to avoid having retries taking place at the gRPC layer and within their own libraries.
客户端不能覆盖服务配置的重试配置，但是，gRPC 客户端库可以禁止所有的重试，这个设计用于使用自己封装实现的重试，避免在 gRPC 层替换重试重试策略

Eventually, retry logic should be taken out of the wrapping libraries, and only exist in gRPC. But allowing the retries to be disabled entirely will make that rollout process easier.
最终，重试逻辑应该从包装库中取出，并且只存在于gRPC中，但是允许完全禁用重试将使替换过程更容易


#### Retry and Hedging Statistics 重试和对冲次数统计

gRPC will treat each retry attempt or hedged RPC as a distinct RPC with regards to the current per-RPC metrics. For example, when an RPC fails with a retryable status code and a retry attempt is made, the original request and the retry attempt will be recorded as two separate RPCs.
就当前的 gRPC 指标策略而言，原始的请求和重试及对冲被认为是不同的请求，如，一个请求失败并且状态是可重试的，并且进行了重试，那么原始的请求和重试的请求将会被记录为两个独立的请求


Additionally, to present a clearer picture of retry attempts, we add three additional per-method metrics:

1. Total number of retry attempts made
2. Total number of retry attempts which failed
3. A histogram of retry attempts made:
    1. The number of retry attempts will be classified into the following buckets:
        1. \>=1, >=2, >=3, >=4, >=5, >=10, >=100, >=1000
    2. Each retry attempt increments the count in exactly bucket. For example:
        1. The 1st retry attempt adds one to the ">=1" count
        2. The 2nd retry attempt adds one to the ">=2" count, leaving the count for ">=1" unchanged.
        3. The 5th through 9th retry attempts each add one to the ">=5" count.
        4. The 10th through 99th retry attempts each add one to the ">=10" count.

For hedged requests, we record the same stats as above, treating the first hedged request as the initial RPC and subsequent hedged requests as retry attempts.

此外，为了更清晰的展示重试，增加了三个额外的指标：
1. 重试的总次数
2. 重试失败的总次数
3. 重试尝试的直方图：
    1. 重试尝试次数将会落在以下桶中：
        1. \>=1, >=2, >=3, >=4, >=5, >=10, >=100, >=1000
    2. 每次重试都会增加桶中的技术，如：
        1. 第一次重试会使 ">=1" 增加1
        2. 第二次重试会使 ">=2" 增加1，但是 ">=1" 不会变
        3. 第五到第九次重试会使 ">=5" 加一
        4. 第十到第99次重试会使 ">=10" 加一

对于对冲请求，记录的指标和上面一样，将第一个被对冲的请求视为初始RPC，将随后的被对冲请求视为重试尝试

### Configuration Language 配置语言

Retry and hedging configuration is set as part of the service config, which is transmitted to the client during DNS resolution. Like other aspects of the service config, retry or hedging policies can be specified per-method, per-service, or per-server name.
重试和对冲配置是服务配置的一部分，在 DNS 解析阶段传递给客户端，和服务配置的其他方面一样，重试和对冲策略可以指定在每个方法，每个服务或这每个服务名

Service owners must choose between a retry policy or a hedging policy. Unless the service owner specifies a policy in the configuration, retries and hedging will not be enabled. The retry policy and hedging policy each have their own set of configuration options, detailed below.
服务所有者必须在重试和对冲策略中选择一个，除非在配置中指定了策略，否则重试和对冲不会生效；重试策略和对冲策略都有自己的一组配置选项，具体如下所示

The parameters for throttling retry attempts and hedged RPCs when failures exceed a certain threshold are also set in the service config. Throttling applies across methods and services on a particular server, and thus may only be configured per-server name.
在服务配置中还设置了在故障超过某个阈值时限制重试尝试和已对冲的 RPC 的参数，节流应用于特定服务器上的方法和服务之间，因此可能只对每个服务器名进行配置


#### Retry Policy 重试策略

This is an example of a retry policy and its associated configuration. It implements exponential backoff with a maximum of four RPC attempts (1 original RPC, and 3 retries), only retrying RPCs when an `UNAVAILABLE` status code is received.
重试策略举例以及相关的配置，实现了指数退避，最大重试次数是4(一次原始请求和三次重试)，只有返回 `UNAVAILABLE` 时才会重试

```
"retryPolicy": {
  "maxAttempts": 4,
  "initialBackoff": "0.1s",
  "maxBackoff": "1s",
  "backoffMultiplier": 2,
  "retryableStatusCodes": [
    "UNAVAILABLE"
  ]
}
```

#### Hedging Policy 对冲策略

The following example of a hedging policy configuration will issue an original RPC, then up to three hedged requests for each RPC, spaced out at 500ms intervals, until either: one of the requests receives a valid response, all fail, or the overall call deadline is reached. Analogously to `retryableStatusCodes` for the retry policy, `nonFatalStatusCodes` determines how hedging behaves when a non-OK response is received.
以下对冲策略举例将发出原始请求，然后每个 RPC最多有三次对冲请求，间隔500ms，直到：收到有效的响应，所有的请求都失败，或者达到请求的最后时间，和 `retryableStatusCodes` 类似，`nonFatalStatusCodes` 根据非ok的响应状态决定对冲行为


```
"hedgingPolicy": {
  "maxAttempts": 4,
  "hedgingDelay": "0.5s",
  "nonFatalStatusCodes": [
    "UNAVAILABLE",
    "INTERNAL",
    "ABORTED"
  ]
}
```

The following example issues four RPCs simultaneously:
下面的示例同时发出四个请求

```
"hedgingPolicy": {
  "maxAttempts": 4,
  "hedgingDelay": "0s",
  "nonFatalStatusCodes": [
    "UNAVAILABLE",
    "INTERNAL",
    "ABORTED"
  ]
}
```

#### Throttling Configuration 节流配置

Throttling configuration applies to all services and methods on a given server, and so can only be set per-server name. The following configuration throttles retry attempts and hedged RPCs when the client's ratio of failures to successes exceeds ~10%.
节流配置对所有请求的所有方法都生效，所以只能对每个服务器名配置，当客户端的失败请求与成功请求比例超过 10% 时将会限制对冲和重试请求

```
"retryThrottling": {
  "maxTokens": 10,
  "tokenRatio": 0.1
}
```

#### Integration with Service Config 与服务配置集成

The retry policy is transmitted to the client through the service config mechanism. The following is what the JSON configuration file would look like:
重试策略通过服务配置机制传输给客户端，以下是 JSON 配置的实例

```
{
  "loadBalancingPolicy": string,

  "methodConfig": [
    {
      "name": [
        {
          "service": string,
          "method": string,
        }
      ],

      // Only one of retryPolicy or hedgingPolicy may be set. If neither is set,
      // RPCs will not be retried or hedged.
      // 重试策略和对冲策略只能设置一个，如果都设置了，那么重试或对冲不会生效

      "retryPolicy": {
        // The maximum number of RPC attempts, including the original RPC.
        //
        // This field is required and must be two or greater.
        // 最大重试次数，包含原始请求，必须是大于等于2
        "maxAttempts": number,

        // Exponential backoff parameters. The initial retry attempt will occur at
        // random(0, initialBackoff). In general, the nth attempt since the last
        // server pushback response (if any), will occur at random(0,
        //   min(initialBackoff*backoffMultiplier**(n-1), maxBackoff)).
        // The following two fields take their form from:
        // https://developers.google.com/protocol-buffers/docs/proto3#json
        // They are representations of the proto3 Duration type. Note that the
        // numeric portion of the string must be a valid JSON number.
        // They both must be greater than zero.
        // 指数退避参数，初始重试会在 random(0, initialBackoff) 之内执行，如果还有重试，
        // 之后的第 n个将会在 random(0, min(initialBackoff*backoffMultiplier**(n-1), maxBackoff))
        // 之间，后面的两个配置必须是符合 proto3 规范的时间范围，数值部分必须是有效的 JSON 数值，必须大于0
        "initialBackoff": string,  // Required. Long decimal with "s" appended 必须，数值，以s为单位
        "maxBackoff": string,  // Required. Long decimal with "s" appended 必须，数值，以s为单位
        "backoffMultiplier": number  // Required. Must be greater than zero. 必须，数值

        // The set of status codes which may be retried.
        //
        // Status codes are specified in the integer form or the case-insensitive
        // string form (eg. [14], ["UNAVAILABLE"] or ["unavailable"])
        //
        // This field is required and must be non-empty.
        // 可以重试的状态码集合，可以是大小写不敏感的状态码或者对应的数值，(如. [14], ["UNAVAILABLE"] or ["unavailable"])
        // 必须，且不能为空
        "retryableStatusCodes": []
      }

      "hedgingPolicy": {
        // The hedging policy will send up to maxAttempts RPCs.
        // This number represents the all RPC attempts, including the
        // original and all the hedged RPCs.
        //
        // This field is required and must be two or greater.
        // 对冲策略最大执行次数，代表所有的请求次数，包括原始请求，必须，且大于等于2
        "maxAttempts": number,

        // The original RPC will be sent immediately, but the maxAttempts-1
        // subsequent hedged RPCs will be sent at intervals of every hedgingDelay.
        // Set this to "0s", or leave unset, to immediately send all maxAttempts RPCs.
        // hedgingDelay takes its form from:
        // https://developers.google.com/protocol-buffers/docs/proto3#json
        // It is a representation of the proto3 Duration type. Note that the
        // numeric portion of the string must be a valid JSON number.
        // 原始的请求将会立即发送，但是其余的对冲请求会每隔指定的延迟发送，设置为0s或者不设置，会立即发送
        // 其余所有请求，必须是 proto3 的时间间隔，数值部分必须是有效的 JSON 数值
        "hedgingDelay": string,

        // The set of status codes which indicate other hedged RPCs may still
        // succeed. If a non-fatal status code is returned by the server, hedged
        // RPCs will continue. Otherwise, outstanding requests will be canceled and
        // the error returned to the client application layer.
        //
        // Status codes are specified in the integer form or the case-insensitive
        // string form (eg. [14], ["UNAVAILABLE"] or ["unavailable"])
        //
        // This field is optional.
        // 表示可以对冲的状态码的集合，如果服务端返回的是非失败的状态码，对冲请求将会继续执行，
        // 除此之外，对冲请求将会取消，错误将会返回给客户端应用层
        // 可以对冲的状态码集合，可以是大小写不敏感的状态码或者对应的数值，(如. [14], ["UNAVAILABLE"] or ["unavailable"])
        // 可以为空
        "nonFatalStatusCodes": []
      }

      "waitForReady": bool,
      "timeout": string,
      "maxRequestMessageBytes": number,
      "maxResponseMessageBytes": number
    }
  ]

  // If a RetryThrottlingPolicy is provided, gRPC will automatically throttle
  // retry attempts and hedged RPCs when the client’s ratio of failures to
  // successes exceeds a threshold.
  // 如果提供了重试呢给节流策略，如果客户端失败请求和成功请求比率超过阈值，gRPC 会自动限制重试和对冲

  // For each server name, the gRPC client will maintain a token_count which is
  // initially set to maxTokens, and can take values between 0 and maxTokens.
  // 对于每个服务名，gRPC会维护初始值为 maxTokens 的 token_count，可以是 0 到 maxTokens
  //
  // Every outgoing RPC (regardless of service or method invoked) will change
  // token_count as follows:
  //
  //   - Every failed RPC will decrement the token_count by 1.
  //   - Every successful RPC will increment the token_count by tokenRatio.
  //
  // 不管调用是什么服务或方法，每个发出的请求都会改变token_count
  //   - 每个失败的请求都会使 token_count 减一
  //   - 每个成功的请求都会是 token_count 加 tokenRatio
  // 
  // If token_count is less than or equal to maxTokens / 2, then RPCs will not
  // be retried and hedged RPCs will not be sent.
  // 如果token_count 小于等于 maxTokens / 2，重试和对冲的请求将不会发送
  "retryThrottling": {
    // The number of tokens starts at maxTokens. The token_count will always be
    // between 0 and maxTokens.
    //
    // This field is required and must be in the range (0, 1000].  Up to 3
    // decimal places are supported
    // tokens 的初始数量为 maxTokens ，token_count 返回在 0 和 maxTokens 之间
    // 必须，且范围在 (0, 1000]，支持小数点后三位
    "maxTokens": number,

    // The amount of tokens to add on each successful RPC. Typically this will
    // be some number between 0 and 1, e.g., 0.1.
    //
    // This field is required and must be greater than zero. Up to 3 decimal
    // places are supported.
    // 每个成功的请求添加的 token 数量，通常在0和1之间
    // 这个参数是必须的，其必须大于-，支持小数点后三位
    "tokenRatio": number
  }
}
```