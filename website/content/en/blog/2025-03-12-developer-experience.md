# Introduction

Over the years, cloud-based software teams have offloaded a lot of functionality from the application layer into the platform and infrastructure layers. For example, authentication and authorization used to be functions inherent in every service, with each service having its own way of checking requester identity and permissions. With the rise in popularity of techniques like mTLS, and tooling combinations like Istio and OPA handling traffic authorization checks, we can centralize those capabilities to prevent disparities in implementation, and benefit from less duplicated code.

The point here isn't to say that this is a bad thing; only that it happened, and when it happened there were inherent trade-offs. This blog posts questions whether or not these trade-offs are always in the best interests of our teams, especially in developer experience. The key word here is "always," and the goal here is ask what teams that are looking for a change can do to mitigate some of these trade-offs.

# Loose Coupling and Lock-In Avoidance

One of the emerging common practices of the cloud-native revolution was to take the notion of a 12-Factor application to what some might consider an extreme. That is, by using bare-bones functionality such as raw TCP requests (in the form of HTTP or gRPC) and vendor agnostic libraries, developers and infrastructure engineers have the freedom to move code as they see fit, from one cloud provider to another, or even one deployment environment to another. Kubernetes isn't working out? That's fine, we'll just move you over to this VM in a data center!

The mindset behind this is that the code should be able to run anywhere, with as few external dependencies as possible. The developers need only to focus on the business logic, and the actual deployment environment shouldn't be a factor.

# Portability Doesn't Mean Portable

While this may sound like a noble endeavor, it gave birth to a number of new problems that the industry seems reluctant to acknowlege - akin to the honeymoon phase of microservice architecture, when we as an industry shuffled under the rug the new challenges that microservices brought when moving away from monoliths.

First and foremost, the application as a whole does still have hard dependencies on the infrastructure; they're just less visible than they would be if they were explicitly written in code. Let's take the above example of offloading authentication and authorization to the infrastructure layer. From a business value perspective, authentication and authorization are still required, regardless of where they're happening. If we take advantage of code portability, and move a service to a different deployment environment - e.g. an edge device, or even a different cloud environment - then we have to stand up new infrastructure functionality to manage that authentication and authorization.

This raises the question - is the code really portable, from a business value point of view? The answer is a pretty obvious no. What's portable is an incomplete set of functionality that's missing essential capabilities. Yes, it will probably run. But there's still work to be done to make it production-ready.

# At the Mercy of the Environment

Additionally, the approach of keeping the code uninformed is not without its holes which, in some cases, can indirectly result in the hard-coding of dependencies that we've tried to avoid.

The example that comes to mind is secrets management. There are several ways to pass secrets into a running pod in a Kubernets cluster, including:

- as environment variables, adhering to the true 12-Factor specification
- as a single mounted file akin to AWS Secrets Manager or Hashicorp Vault
- as a directory of mounted files, with each file name corresponding to the secret key, and the content of the file being the secret value

Each of these approaches has direct code implications that can kill portability. In fact, even if the same secrets lookup approach is used, the results are not always the same. For example, when leveraging a single mounted file, the format of that file must match when moving from one tool or environment to another: `key=value` vs `{ "key": "value" }`.

# It Really Did Work on My Computer

The result is that it becomes harder for local development to match the deployment environment. In this case, harder means there are understood gaps, or there is additional tooling needed and the decades old paradigm of local development may fall short.

This gap is reinforced by the goal of keeping the code uninformed of the environment in which it will be deployed. While developers can install tools like Minikube, KinD, or K3s, the practice of keeping production tooling transparent to developers - CSIs, event handlers, network proxies, secrets stores, etc. - means they are unlikely to have those tools installed locally. Thus the gap in parity between dev and prod.

Even more confounding to the developer - as capabilities were offloaded to the infrastructure, visibility into those capabilities was subsequently removed from the developer purvue as well. Centralized log aggregation and distributed trace metrics are often unavailable without the help of an infrastructure team to set them up and configure them, giving access where applicable. Admittedly, this a is demonstrably better approach than the ancient practice of everyone SSH-ing into different boxes and watching logs as they happen. But it also takes the developer out of the code and puts them into a realm they are - by design - not meant to be familiar with, which is the underlying infrastructure.

# Platforms to the Rescue?

The rise of platform engineering has made internal development platforms (IDPs) a more popular solution for setting up development environments. This is, at the very least, a laudable direction for software engineering teams to take in closing the gap, and creating a positive developer experience.

The concern with this approach is that it significantly increases organizational and tooling overhead. It's 100% plausible that this investment is worth it for enterprise teams. However the investment may not be feasible for smaller teams that need to build and more quickly in order to justify their next paycheck. When every development dollar has to be focused on customer value and revenue generation, the possibility of a centralized platform team and an IDP tends to go out the window.

Moreover, it puts developers that use the IDP at the mercy of said IDP. If there are network restrictions in gaining access to the IDP, then there will be times when the developer could work on their code, but can't. Usually this is solved with a VPN, but there are offline or low bandwidth cases (e.g. in transit) when productivity is lost. Plenty of teams can afford that developer downtime. But not all.

This also puts the dev-prod parity burden on the platform team to maintain. An efficient platform team will be in constant communication with both the infrastructure teams and the development teams and will always be up-to-date on tooling changes and capability enhancements - something that is extremely hard to keep up with. Yes, there are teams that can pull this off, but certainly not all.

# Embrace It

One of the CNCF projects that is focused on closing this gap, while still empowering the developer, is Dapr. Instead of rejecting this gap between the developer and the environment, Dapr acknowleges and embraces this reality as, in some cases, unavoidable.

When the notion of code portability falls short, it can usually be solved with an abstraction layer, which what Dapr offers. Developers aren't forced to directly implement code that indirectly depends on infrastructure tooling or formatting. Rather, they call abstracted functions in Dapr that are then adapted by the infrastructure teams to match the deployment environment's tooling.

Does this solve all of the problems or gaps in the developer experience? Probably not, but it's a start. Infrastructure teams are able to move code and tools freely as business needs require, and the abstraction layer that Dapr provides allows them to meet the developers in the middle. Developers don't code for the environment, but their code is less likely to break when things inevitably change.

# Reject It

Another CNCF project, NATS, takes a very different approach. It moves the communications paradigm directly into the code. So whether the developer is using NATS for streaming, or transactional request-response, the developer is aware of how their service-to-service and 1:M interactions are taking place, and it's written using language-specific SDKs.

This doesn't solve all of the problems listed above, but it does solve quite a few. First and foremost, there's no secret as to what's happening in the environment, and the developer has a clear model that can achieve parity for running service-to-service and 1:M communication locally. Subsequently, if the developer chooses to use (and the infrastructure supports) the data storage capabilities in NATS, then even more of the code can reflect what's happening in the environment. Finally, NATS can be used to solve a lot of the common capabilities that has been offloaded to different infrastructure tools, such as authentication and authorization, centralized logging, and trace metrics. This gives the developer a single tool to understand in the broader scope of the infrastructure.

The irony is that one of the main selling points of NATS is that, as long as the code can reach a server in the NATS cluster, even if it's a leaf-node at the edge, it is indeed truly portable from a communications standpoint. Of course, other factors such as secrets injection would still have to be solved for.

If this sounds like a monolithic approach to infrastructure tooling, that's because it is. And it comes with the same tooling risks that one would expect in building around a single project - mainly, if that project goes away, it all falls apart. Additionally, at the code level, this requires the constant maintenance and upkeep of language-specific SDKs, which can potentially see feature lag, depending on which languages are more popular at any given time. SDK maintenance is a ton of work, and brings with it an exponential increase in required documentation.

So yes, it's a risk, but for some dev teams it may be worth it.

All that said, cloud providers have been pushing this paradigm for years, with no hint of stopping. And in that time, some (but definitely not all) dev teams have embraced cloud provider SDKs as a way to interact with their production environments. Following loose-coupling practices in the code, such as adding abstraction in the form of interfaces and service layers, makes the inevitable code changes less painful on the developer if/when things do change.

# Is WASM the Future?

One of the promises of WebAssembly (WASM) is that, in the future, the WASM module approach will solve this once and for all. WASM modules are purported to take the direct 