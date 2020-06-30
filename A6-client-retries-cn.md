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
对于节流策略而言

##### Validation of retryThrottling

If `retryThrottling` is specified in a service config, the following validation rules apply:
1. `maxTokens` MUST be specified and MUST be a JSON integer value in the range (0, 1000].
2. `tokenRatio` MUST be specified and MUST be a JSON floating point greater than 0. Decimal places beyond 3 are ignored. (eg. 0.5466 is treated as 0.546.)

#### Pushback

Servers may explicitly pushback by setting metadata in their response to the client. The pushback can either tell the client to retry after a given delay or to not retry at all. If the client has already exhausted its `maxAttempts`, the call will not be retried even if the server says to retry after a given delay.

Pushback may also be received to a hedged request. If the pushback says not to retry, no further hedged requests will be sent. If the pushback says to retry after a given delay, the next hedged request (if any) will be issued after the given delay has elapsed.

A new metadata key, `"grpc-retry-pushback-ms"`, will be added to support server pushback. The value is to be an ASCII encoded signed 32-bit integer with no unnecessary leading zeros that represents how many milliseconds to wait before sending a retry. If the value for pushback is negative or unparseble, then it will be seen as the server asking the client not to retry at all.

When a client receives an explicit pushback response from a server, and it is appropriate to retry the RPC, it will retry after exactly that delay.  For subsequent retries, the delay period will be reset to the `initialBackoff` setting and scale according to the [Exponential Backoff](#exponential-backoff) section above, unless an explicit pushback response is received again.

#### Limits on Retries and Hedges

`maxAttempts` in both `retryPolicy` and `hedgingPolicy` have, by default, a client-side maximum value of 5. This client-side maximum value can be changed by the client through the use of channel arguments. Service owners may specify a higher value for these parameters, but higher values will be treated as equal to the maximum value by the client implementation. This mitigates security concerns related to the service config being transferred to the client via DNS.

#### Summary of Retry and Hedging Logic
There are five possible types of server responses. The list below enumerates the behavior of retry and hedging policies for each type of response. In all cases, if the maximum number of retry attempts or the maximum number of hedged requests is reached, no further RPCs are sent. Hedged RPCs are returned to the client application when all outstanding and pending requests have either received a response or been canceled.

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

![State Diagram](A6_graphics/StateDiagram.png)

[Link to SVG file](A6_graphics/StateDiagram.svg)

### Retry Internals

#### Where Retries Occur

The retry policy will be implemented in-between the channel and the load balancing policy. That way every retry gets a chance to be sent out on a different subchannel than it originally failed on.

![Where Retries Occur](A6_graphics/WhereRetriesOccur.png)

[Link to SVG file](A6_graphics/WhereRetriesOccur.svg)

#### When Retries are Valid

In certain cases it is not valid to retry an RPC. These cases occur when the RPC has been *committed*, and thus it does not make sense to perform the retry.

An RPC becomes *committed* in two scenarios:

1. The client receives [Response-Headers](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md#responses).
2. The client’s outgoing message has overflowed the gRPC client library’s buffer.

The reasoning behind the first scenario is that the Response-Headers include initial metadata from the server. The metadata (or its absence) it is transmitted to the client application. This may fundamentally change the state of the client, so we cannot safely retry if a failure occurs later in the RPC’s life.

gRPC servers should delay the Response-Headers until the first response message or until the application code chooses to send headers. If the application code closes the stream with an error before sending headers or any response messages, gRPC servers should send the error in Trailers-Only.

To clarify the second scenario, we define an *outgoing message* as everything the client sends on its connection to the server. For unary and server streaming calls, the outgoing message is a single message. For client and bidirectional streaming calls, the outgoing message is the entire message stream issued by the client after opening the connection. The gRPC client library buffers outgoing messages, and as long as the entirety of the outgoing message is in the buffer, it can be resent and retried. But as soon as the outgoing message grows too large to buffer, the gRPC client library cannot replay the entire stream of messages, and thus retries are not valid.

#### Memory Management (Buffering)

The gRPC client library will support application-configured limits for the amount of memory used for retries. It is suggested that the client sets `retry_buffer_size_in_bytes` to limit the total amount of memory used to buffer retryable or hedged RPCs. The client should also set `per_rpc_buffer_limit_in_bytes` to limit the amount of memory used by any one RPC (to prevent a single large RPC from using the whole buffer and preventing retries of subsequent smaller RPCs). These limits are configured by the client, rather than coming from the service config.

RPCs may only be retried when they are contained in the buffer. New RPCs which do not fit in the available buffer space (either due to the total available buffer space, or due to the per-RPC limit) will not be retryable, but the original RPC will still be sent.

After the RPC response has been returned to the client application layer, the RPC is removed from the buffer.

Client streaming RPCs will take up additional buffer space with each subsequent message, so additional buffering policy is needed. When the application sends a message on an RPC that causes the RPC to exceed the buffer limit, the RPC becomes committed, meaning that we choose one attempt to continue and stop all others.

For retriable RPCs, when an RPC becomes committed, the client will continue with the currently in-flight attempt but will not make any subsequent attempts, even if the current attempt fails with a retryable status and there are retry attempts remaining.

For hedged RPCs, when an RPC becomes committed, the client will continue the currently in-flight attempt on which the maximum number of messages have already been sent. All other currently in-flight attempts will be immediately cancelled, and no subsequent attempts will be started.

Once we have committed to a particular attempt, any messages that have already been sent on the that attempt can be immediately freed from the buffer, and each subsequent message that is replayed can be freed as soon as it is sent. The new message sent by the application (which caused the RPC to exceed the buffer limit) will never be added to the buffer in the first place; it will stay pending until all previous messages have been replayed, and then it will be sent immediately without buffering.

When an RPC is evicted from the buffer, pending hedged requests should be canceled immediately. Implementations must avoid potential scheduling race conditions when canceling pending requests concurrently with incoming responses to already sent hedges, and ensure that failures are relayed to the client application logic when no more hedged requests will be possible and all outstanding requests have returned.

#### Transparent Retries

RPC failures can occur in three distinct ways:

1. The RPC never leaves the client.
2. The RPC reaches the server, but has never been seen by the server application logic.
3. The RPC is seen by the server application logic, and fails.

![Where RPCs Fail](A6_graphics/WhereRPCsFail.png)

[Link to SVG file](A6_graphics/WhereRPCsFail.svg)

The last case is handled by the configurable retry policy that is the main focus of this document. The first two cases are retried automatically by the gRPC client library, **regardless** of the retry configuration set by the service owner. We are able to do this because these request have not made it to the server application logic, and thus are always safe to retry.

In the first case, in which the RPC never leaves the client, the client library can transparently retry until a success occurs, or the RPC's deadline passes.

If the RPC reaches the gRPC server library, but has never been seen by the server application logic (the second case), the client library will immediately retry it once. If this fails, then the RPC will be handled by the configured retry policy. This extra caution is needed because this case involves extra load on the wire.

Since retry throttling is designed to prevent server application overload, and these transparent retries do not make it to the server application layer, they do not count as failures when deciding whether to throttle retry attempts.

Similarly, transparent retries do not count toward the limit of configured RPC attempts (`maxAttempts`).

![State Diagram](A6_graphics/transparent.png)

[Link to SVG file](A6_graphics/transparent.svg)

#### Exposed Retry Metadata

Both client and server application logic will have access to data about retries via gRPC metadata. Upon seeing an RPC from the client, the server will know if it was a retry, and moreover, it will know the number of previously made attempts. Likewise, the client will receive the number of retry attempts made when receiving the results of an RPC.

The header name for exposing the metadata will be `"grpc-previous-rpc-attempts"` to give clients and servers access to the attempt count. This value represents the number of preceding retry attempts. Thus, it will not be present on the first RPC, will be 1 for the second RPC, and so on. The value for this field will be an integer.

#### Disabling Retries

Clients cannot override retry policy set by the service config. However, retry support can be disabled entirely within the gRPC client library. This is designed to enable existing libraries that wrap gRPC with their own retry implementation (such as Veneer Toolkit) to avoid having retries taking place at the gRPC layer and within their own libraries.

Eventually, retry logic should be taken out of the wrapping libraries, and only exist in gRPC. But allowing the retries to be disabled entirely will make that rollout process easier.

#### Retry and Hedging Statistics

gRPC will treat each retry attempt or hedged RPC as a distinct RPC with regards to the current per-RPC metrics. For example, when an RPC fails with a retryable status code and a retry attempt is made, the original request and the retry attempt will be recorded as two separate RPCs.

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

### Configuration Language

Retry and hedging configuration is set as part of the service config, which is transmitted to the client during DNS resolution. Like other aspects of the service config, retry or hedging policies can be specified per-method, per-service, or per-server name.

Service owners must choose between a retry policy or a hedging policy. Unless the service owner specifies a policy in the configuration, retries and hedging will not be enabled. The retry policy and hedging policy each have their own set of configuration options, detailed below.

The parameters for throttling retry attempts and hedged RPCs when failures exceed a certain threshold are also set in the service config. Throttling applies across methods and services on a particular server, and thus may only be configured per-server name.

#### Retry Policy

This is an example of a retry policy and its associated configuration. It implements exponential backoff with a maximum of four RPC attempts (1 original RPC, and 3 retries), only retrying RPCs when an `UNAVAILABLE` status code is received.

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

#### Hedging Policy

The following example of a hedging policy configuration will issue an original RPC, then up to three hedged requests for each RPC, spaced out at 500ms intervals, until either: one of the requests receives a valid response, all fail, or the overall call deadline is reached. Analogously to `retryableStatusCodes` for the retry policy, `nonFatalStatusCodes` determines how hedging behaves when a non-OK response is received.

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

#### Throttling Configuration
Throttling configuration applies to all services and methods on a given server, and so can only be set per-server name. The following configuration throttles retry attempts and hedged RPCs when the client's ratio of failures to successes exceeds ~10%.

```
"retryThrottling": {
  "maxTokens": 10,
  "tokenRatio": 0.1
}
```

#### Integration with Service Config

The retry policy is transmitted to the client through the service config mechanism. The following is what the JSON configuration file would look like:

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

      "retryPolicy": {
        // The maximum number of RPC attempts, including the original RPC.
        //
        // This field is required and must be two or greater.
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
        "initialBackoff": string,  // Required. Long decimal with "s" appended
        "maxBackoff": string,  // Required. Long decimal with "s" appended
        "backoffMultiplier": number  // Required. Must be greater than zero.

        // The set of status codes which may be retried.
        //
        // Status codes are specified in the integer form or the case-insensitive
        // string form (eg. [14], ["UNAVAILABLE"] or ["unavailable"])
        //
        // This field is required and must be non-empty.
        "retryableStatusCodes": []
      }

      "hedgingPolicy": {
        // The hedging policy will send up to maxAttempts RPCs.
        // This number represents the all RPC attempts, including the
        // original and all the hedged RPCs.
        //
        // This field is required and must be two or greater.
        "maxAttempts": number,

        // The original RPC will be sent immediately, but the maxAttempts-1
        // subsequent hedged RPCs will be sent at intervals of every hedgingDelay.
        // Set this to "0s", or leave unset, to immediately send all maxAttempts RPCs.
        // hedgingDelay takes its form from:
        // https://developers.google.com/protocol-buffers/docs/proto3#json
        // It is a representation of the proto3 Duration type. Note that the
        // numeric portion of the string must be a valid JSON number.
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
  //
  // For each server name, the gRPC client will maintain a token_count which is
  // initially set to maxTokens, and can take values between 0 and maxTokens.
  //
  // Every outgoing RPC (regardless of service or method invoked) will change
  // token_count as follows:
  //
  //   - Every failed RPC will decrement the token_count by 1.
  //   - Every successful RPC will increment the token_count by tokenRatio.
  //
  // If token_count is less than or equal to maxTokens / 2, then RPCs will not
  // be retried and hedged RPCs will not be sent.
  "retryThrottling": {
    // The number of tokens starts at maxTokens. The token_count will always be
    // between 0 and maxTokens.
    //
    // This field is required and must be in the range (0, 1000].  Up to 3
    // decimal places are supported
    "maxTokens": number,

    // The amount of tokens to add on each successful RPC. Typically this will
    // be some number between 0 and 1, e.g., 0.1.
    //
    // This field is required and must be greater than zero. Up to 3 decimal
    // places are supported.
    "tokenRatio": number
  }
}
```
