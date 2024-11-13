# crossplane-terraform
Notes on the interaction of Crossplane with Terraform 

In my talks about Crossplane I often get questions about the nature of the Crossplane Terraform integration, since Crossplane's Terrajet/Upjet generated Providers use Terraform under the hood. So this is a incomplete composition of some links and TLDRs about the topic.

## State

If Crossplane uses Terraform as a basis for the Provider generation via Upjet, how does it handle state?

https://www.reddit.com/r/Terraform/comments/1asr122/terraform_vs_crossplane/

https://spacelift.io/blog/crossplane-vs-terraform:

It reconciles state independent of other recources, really just based on the manifests deployed to the management cluster / control plane.


deep dive

https://blog.crossplane.io/deep-dive-terrajet-part-i/

https://blog.crossplane.io/deep-dive-terrajet-part-ii/

> "Applying this to our controller, it will need to watch the current state of our external resource and try to bring it to the desired state. Here comes the tricky part, how should a controller that keeps running can bring the external resource into the desired state with Terraform CLI? Should we simply call terraform apply periodically and let it do the rest? Yeah, that could be a good POC :) However, for something real, we wouldn’t want a controller that keeps calling long-running terraform apply’s on our infrastructure resource no matter it is already up to date or not. Following the Kubernetes controller pattern, we want our controller to watch the state of our resource before taking any action. In other words, observe the current state and try to move it to the desired state only if they do not match."


__TLDR: Terrajet/Upjet implements the `ExternalConnector` implementing Crossplane's `ExternalClient` interface, that uses Terraform CLI under the hood.__

> "In Crossplane, there is always a one-to-one mapping between a CR and an external resource."

--> __This means, that there's only one resource defined in a Pod local `.tfstate` file, which will not even be persistet - but just be generated from the Composite Resource state, which is managed by K8s' single source of truth etcd.__

For Observing the state of a resource, the `ExternalConnector` makes heavy use of `terraform plan`.



> Terraform keeps the last applied state in a local .tfstate file and does a refresh to update it with the current state prior to any operation. According to terraform documentation, this state file is used to:

    Map real-world resources to your configuration
    Keep track of metadata
    Improve performance for large infrastructures

In Crossplane, there is always a one-to-one mapping between a CR and an external resource. This means that there will always be one resource in our .tfstate hence we can ignore the last use case. For mapping real word resources, Crossplane has the External Name concept. For tracking metadata, we can use annotations or labels of our CRs. So, we can indeed get rid of the management of a .tfstate file by translating pieces into Crossplane/Kubernetes realm.

Once we can uniquely identify an external resource using its External Name, we can just observe its current state by “refreshing”. The desired state is already available in our CR, hence we can compare and figure out whether the resource is up to date or not by invoking a plan command. More details on the implementation of this coming in the next blog post. All we need to know for now is, there is no .tfstate file persisted somewhere, rather, we are just (re)building it using the information in our CR whenever it is needed.



--> Terrajet to Upjet: https://github.com/crossplane/terrajet/issues/308

https://blog.upbound.io/first-official-providers


## Cloud Provider API polling?

https://blog.crossplane.io/deep-dive-terrajet-part-ii/

The Terrajet/Upjet provider's `ExternalConnector`'s Observe method is is implemented in a way, to reduce or even avoid unnecessary `terraform plan`s.



