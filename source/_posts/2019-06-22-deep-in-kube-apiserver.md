---
title: kube-apiserver代码解读
date: 2019-06-22 16:28:37
tags: code，k8s
---





## 1 关键数据结构



### 1.1 `Master`



`Master`包含了`kube-apiserver`中的所有参数

```go
// Master contains state for a Kubernetes cluster master/api server.
type Master struct {
	GenericAPIServer *genericapiserver.GenericAPIServer

	ClientCARegistrationHook ClientCARegistrationHook
}
```

其中最关键的元素是`GenericAPIServer *genericapiserver.GenericAPIServer`。

### 1.2 `GenericAPIServer`

<!-- more -->

```go
// GenericAPIServer contains state for a Kubernetes cluster api server.
type GenericAPIServer struct {
   // discoveryAddresses is used to build cluster IPs for discovery.
   discoveryAddresses discovery.Addresses

   // LoopbackClientConfig is a config for a privileged loopback connection to the API server
   LoopbackClientConfig *restclient.Config

   // minRequestTimeout is how short the request timeout can be.  This is used to build the RESTHandler
   minRequestTimeout time.Duration

   // ShutdownTimeout is the timeout used for server shutdown. This specifies the timeout before server
   // gracefully shutdown returns.
   ShutdownTimeout time.Duration

   // legacyAPIGroupPrefixes is used to set up URL parsing for authorization and for validating requests
   // to InstallLegacyAPIGroup
   legacyAPIGroupPrefixes sets.String

   // admissionControl is used to build the RESTStorage that backs an API Group.
   admissionControl admission.Interface

   // SecureServingInfo holds configuration of the TLS server.
   SecureServingInfo *SecureServingInfo

   // ExternalAddress is the address (hostname or IP and port) that should be used in
   // external (public internet) URLs for this GenericAPIServer.
   ExternalAddress string

   // Serializer controls how common API objects not in a group/version prefix are serialized for this server.
   // Individual APIGroups may define their own serializers.
   Serializer runtime.NegotiatedSerializer

   // "Outputs"
   // Handler holds the handlers being used by this API server
   Handler *APIServerHandler

   // listedPathProvider is a lister which provides the set of paths to show at /
   listedPathProvider routes.ListedPathProvider

   // DiscoveryGroupManager serves /apis
   DiscoveryGroupManager discovery.GroupManager

   // Enable swagger and/or OpenAPI if these configs are non-nil.
   swaggerConfig *swagger.Config
   openAPIConfig *openapicommon.Config

   // PostStartHooks are each called after the server has started listening, in a separate go func for each
   // with no guarantee of ordering between them.  The map key is a name used for error reporting.
   // It may kill the process with a panic if it wishes to by returning an error.
   postStartHookLock      sync.Mutex
   postStartHooks         map[string]postStartHookEntry
   postStartHooksCalled   bool
   disabledPostStartHooks sets.String

   preShutdownHookLock    sync.Mutex
   preShutdownHooks       map[string]preShutdownHookEntry
   preShutdownHooksCalled bool

   // healthz checks
   healthzLock    sync.Mutex
   healthzChecks  []healthz.HealthzChecker
   healthzCreated bool

   // auditing. The backend is started after the server starts listening.
   AuditBackend audit.Backend

   // Authorizer determines whether a user is allowed to make a certain request. The Handler does a preliminary
   // authorization check using the request URI but it may be necessary to make additional checks, such as in
   // the create-on-update case
   Authorizer authorizer.Authorizer

   // enableAPIResponseCompression indicates whether API Responses should support compression
   // if the client requests it via Accept-Encoding
   enableAPIResponseCompression bool

   // delegationTarget is the next delegate in the chain. This is never nil.
   delegationTarget DelegationTarget

   // HandlerChainWaitGroup allows you to wait for all chain handlers finish after the server shutdown.
   HandlerChainWaitGroup *utilwaitgroup.SafeWaitGroup

   // The limit on the request body size that would be accepted and decoded in a write request.
   // 0 means no limit.
   maxRequestBodyBytes int64
}
```

其中最关键的元素是`Handler *APIServerHandler`。







### 1.3 `APIServerHandlers`



```go
// APIServerHandlers holds the different http.Handlers used by the API server.
// This includes the full handler chain, the director (which chooses between gorestful and nonGoRestful,
// the gorestful handler (used for the API) which falls through to the nonGoRestful handler on unregistered paths,
// and the nonGoRestful handler (which can contain a fallthrough of its own)
// FullHandlerChain -> Director -> {GoRestfulContainer,NonGoRestfulMux} based on inspection of registered web services
type APIServerHandler struct {
   // FullHandlerChain is the one that is eventually served with.  It should include the full filter
   // chain and then call the Director.
   FullHandlerChain http.Handler
   // The registered APIs.  InstallAPIs uses this.  Other servers probably shouldn't access this directly.
   GoRestfulContainer *restful.Container
   // NonGoRestfulMux is the final HTTP handler in the chain.
   // It comes after all filters and the API handling
   // This is where other servers can attach handler to various parts of the chain.
   NonGoRestfulMux *mux.PathRecorderMux

   // Director is here so that we can properly handle fall through and proxy cases.
   // This looks a bit bonkers, but here's what's happening.  We need to have /apis handling registered in gorestful in order to have
   // swagger generated for compatibility.  Doing that with `/apis` as a webservice, means that it forcibly 404s (no defaulting allowed)
   // all requests which are not /apis or /apis/.  We need those calls to fall through behind goresful for proper delegation.  Trying to
   // register for a pattern which includes everything behind it doesn't work because gorestful negotiates for verbs and content encoding
   // and all those things go crazy when gorestful really just needs to pass through.  In addition, openapi enforces unique verb constraints
   // which we don't fit into and it still muddies up swagger.  Trying to switch the webservices into a route doesn't work because the
   //  containing webservice faces all the same problems listed above.
   // This leads to the crazy thing done here.  Our mux does what we need, so we'll place it in front of gorestful.  It will introspect to
   // decide if the route is likely to be handled by goresful and route there if needed.  Otherwise, it goes to PostGoRestful mux in
   // order to handle "normal" paths and delegation. Hopefully no API consumers will ever have to deal with this level of detail.  I think
   // we should consider completely removing gorestful.
   // Other servers should only use this opaquely to delegate to an API server.
   Director http.Handler
}
```

其中最关键的数据结构是`GoRestfulContainer *restful.Container`







