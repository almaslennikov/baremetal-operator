# Operator-SDK Query API Design Doc

## Goal

In order to build a functional operator, the ability to observe state is a must. For e.g the etcd-operator needs to know the number of live etcd pods so that it can reconcile the etcd cluster based on that. The operator-sdk needs to provide an API to Query(`Get`, `List`) objects that would allow an operator to observe the state of its application.

## Background

Client-go provides APIs for querying objects in the standard Kubernetes groups via `Get` and `List`.
However, each client is specific to an object’s APIVersion and Kind which makes the API too verbose.
For example, to get a specific Deployment object:
- Create a Kubernetes clientset that implements kubernetes.Interface.
- Call `kubeclient.AppsV1().Deployments("default").Get("name", v1.GetOptions{})`

To retrieve a different object like a Pod, the above API would change to:
`kubeclient.CoreV1().Pods("default").Get("name", v1.GetOptions{})`

In addition Custom Resources also require their own clients that need to be generated by the users themselves via code-generation.

To avoid these issues the SDK can provide a more generic API to query objects that would work for all object types.

## Proposal

To simplify the retrieval of Kubernetes objects, the SDK can use a dynamic resource client to provide an API for `Get()` and `List()` that works for all object APIVersions and Kinds.

## API

### Get()

```Go
// Get gets the Kubernetes object and then unmarshals the retrieved data into the "into" object.
// "into" is a Kubernetes runtime.Object that must have
// "Kind" and "APIVersion" specified in its "TypeMeta" field
// and "Name" and "Namespace" specified in its "ObjectMeta" field.
// Those are used to construct the underlying resource client.
// "opts" configures the Get operation.
func Get(into sdkTypes.Object, opts ...GetOption) error
```

Get() accepts GetOption as variadic functional parameters. In this way, we follow the open-close principle
which allows us to extend the Get API with unforeseen parameters without modifying the API itself. See the reference section on how the options are implemented.

Example Usage:
```Go
// meta_v1 "k8s.io/apimachinery/pkg/apis/meta/v1"

d := &apps_v1.Deployment{
    TypeMeta: meta_v1.TypeMeta{
        Kind:       "Deployment",
        APIVersion: "apps/v1",
    }
    ObjectMeta: meta_v1.ObjectMeta{
        Name:      "example",
        Namespace: "default",
    }
}
// Get with default options
err := sdk.Get(d)
// Get with custom options
o := &meta_v1.GetOptions{ResourceVersion: "0"}
err := sdk.Get(d, sdk.WithGetOptions(o))
```

### List()

```Go
// List gets a list of Kubernetes object and then unmarshals the retrieved data into the "into" object.
// "namespace" indicates which Kubernetes namespace to look for the list of Kubernetes objects.
// "into" is a sdkType.Object that must have
// "Kind" and "APIVersion" specified in its "TypeMeta" field
// Those are used to construct the underlying resource client.
// "opts" configures the List operation.
func List(namespace string, into sdkTypes.Object, opts ...ListOption) error
```

List() accepts ListOptions similar to GetOptions. See the reference section for implementation details.

Example usage:

```Go
// meta_v1 "k8s.io/apimachinery/pkg/apis/meta/v1"
// apps_v1 "k8s.io/api/apps/v1"

dl := &apps_v1.DeploymentList{
    TypeMeta: meta_v1.TypeMeta{
        Kind:       "Deployment",
        APIVersion: "apps/v1",
    }
}
// List with default options
err = sdk.List("default", dl)
// List with custom options
labelSelector := getLabelSelector("app", "dev")
o := &meta_v1.ListOptions{LabelSelector: labelSelector}
err = sdk.List("default", dl, op.WithListOptions(o))
```



## Reference:

### GetOptions:

```Go
// GetOp wraps all the options for Get().
type GetOp struct {
    metaGetOptions *meta_v1.GetOptions
}

// GetOption configures GetOp.
type GetOption func(*GetOp)


// WithGetOptions sets the meta_v1.GetOptions for the Get() operation.
func WithGetOptions(metaGetOptions *meta_v1.GetOptions) GetOption {
    return func(op *GetOp) {
        op.metaGetOptions = metaGetOptions
    }
}
```

### ListOptions:

```Go
// ListOp wraps all the options for List.
type ListOp struct {
    metaListOptions *metav1.ListOptions
}

// ListOption configures ListOp.
type ListOption func(*ListOp)

// WithListOptions sets the meta_v1.ListOptions for
// the List() operation.
func WithListOptions(metaListOptions *meta_v1.ListOptions) ListOption {
    return func(op *ListOp) {
        op.metaListOptions = metaListOptions
    }
}
```