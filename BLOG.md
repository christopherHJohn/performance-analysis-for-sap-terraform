# A performance analysis for SAP related Terraform providers

<img src="./img/cover_image.jpg" width=250px>

When looking for recommendations for Terraform state design, one will often see advice like _"separate rarely changing and frequently changing elements"_ or more geneal _"keep states small"_. This is useful advice when looking at the topic from a high level, but these are no concrete numbers one can use as a reference point. This also leads to one question often being unanswered: _"How many resources are considered a small state?"_

For the SAP related providers, there are differences of more than 200 times in the duration it takes to refresh different resources. This changes what can be considered small for a state and moves the question from the resource count alone to a question about the combination of resource type and resource count.

The goal of this blog is to provide some insight into which resources to look out for when designing scripts in regards to the SAP related provider and to give an estimate for how long state files will take before even running them the first time.

---

- [Performance of resources](#performance-of-resources)
    - [Results](#results)
    - [Why are entitlements so slow?](#why-are-entitlements-so-slow)
    - [Effect of Parallelism](#effect-of-parallelism)
- [What is a small state?](#what-is-a-small-state)
    - [Results](#results-1)
    - [Parallelism](#parallelism)
    - [Introducing Dependencies](#introducing-dependencies)
    - [Dont forget the init](#dont-forget-the-init)
- [Summary](#summary)

---

⚠️ Lets start with an exclaimer:

_The refresh time of resources is heavily dependent on the speed at which the target system can process the request as well as the general network speed. Even while writing this blog, I experienced huge differnces from time to time. While most of these differences amounted to only around 20-30%, when comparing the averages of runs,  comparing single resources can show differences of more than 30 times. To make sure the results shown later are as accurate and comparable as possible, some scripts were executed multiple times over the course of multiple days. But even with these precautions, treat the shown values not as fixed, but as reference points._ 

---

## Performance of resources
Let's start by having a look at the performance of the different resources.

To do this part of the analysis, a script was created containing the SAP providers for [BTP](https://registry.terraform.io/providers/SAP/btp/1.26.0), [SCC](https://registry.terraform.io/providers/SAP/scc/1.6.0) and [SCI](https://registry.terraform.io/providers/SAP/sap-cloud-identity-services/0.7.0-beta1) as well as the [Cloud Foundry](https://registry.terraform.io/providers/cloudfoundry/cloudfoundry/1.18.0) provider. The specific resources included:

- **BTP**: Directory, Subaccount, Entitlement, Environment, Role, Role Collection, Service Instance, Subscription,  Generic Destination, Trust Configuration
- **CF**: Space, Service Instance
- **SCC**: Subaccount, Mapping, Mapping Resource
- **SCI**: Group Base, User, Application

Each resource was included once in the script. After the project was initialised and applied, a terraform plan in refresh only mode was used:‚

```
TF_LOG=TRACE terraform plan -refresh-only > tf.log 2>&1
```

_Refresh only mode is not necessarily required as the performance is mostly the same between both modes._

In the resulting log file, the refresh time of an resource can then be checked by comparing the timestamp of the following two elements. For this case, this would amount to a refresh time of 257ms.

```
sci_group_base.group: Refreshing state... [id=d8398e59-49d9-4745-9437-2e2f6114174a]
2026-09-15T19:06:14.045+0200 [TRACE] GRPCProvider.v6: ReadResource
[...]
2026-09-15T19:06:14.302+0200 [TRACE] vertex "sci_group_base.group": visit complete
```

### Results
The described setup was executed 100 times. The resulting refresh times showed the following ranges for the different resources of the providers:
- **BTP** = 1000-5000ms
- **CF** = 50-200ms
- **SCC** = 100-250ms
- **SCI** =  200-300ms

But as we all know, there is always an exception to the rule. In this case, this exception is the `btp_subaccount_entitlement` resource with a whopping **13-16 seconds** of runtime on average.

Another deviation from this rule is the `scc_subaccount` resource with around 1500ms on average.

As a side note before we continue. For the BTP provider the times can be even further divided with `btp_directory`, `btp_subaccount_service_instance` and `btp_subaccount_subscription` taking around 2500-5000ms, while all other tested resources took around 1000-2000ms.

Lastly, note that Cloud Foundry providers instances are around 60 times faster then their BTP counterpart. So when a lot of instances are required, switching to CF instances may be beneficial.

### Why are entitlements so slow?
With this difference in performance between entitlements and the other BTP resources, one will likely ask _"Why are entitlements so slow?"_.

Sadly, I can't give an answer to this. Having a look into the BTP cli, which uses the same API as the Terraform provider, the cli itself falls in the 1-5s estimate with around 4 seconds on average.

This can be checked via the following command and looking into the verbose output:

```
btp --verbose list accounts/entitlement
```

So either the Terraform provider uses a special endpoint to check these values or the additional processing of the API response that Terraform needs to do takes a lot of time.

### Effect of Parallelism
With the general results out of the way, let’s look at one more phenomenon that I found while checking the parallelism setting for the second chapter. 

First a short description of parallelism: The argument `parallelism=n` allows one to enter a value n (Default = 10), which defines how many operations can be performed at once. Meaning with n=1 only resource at the time can be refreshed, while with n=10 a maximum of 10 resources can be refreshed at once. This is of course limited by (1) the speed of the device on which the command is executed and (2) the API limitations of the target system.

So what do we expect? Parallelism 1 is slower than parallelism 10, which itself is slower than parallelism 20. With that out of the way, one would expect that the chapter can eb closed, which is also what I expected, before checking the numbers for the second part of the anylsis.

When looking at the CF provider, for example, we can see exactly what we expect (running 50 space resources):

| Parallelism | Time (s) | Refresh time per resource (ms) |
| ------------- | ---: | ---: |
| 1 | 9.2 | 146.1 | 
| 5 | 3.8 | 157.9 | 
| 10 | 2.1 | 188.2 | 
| 20 | 1.7 | 204.6 | 
| 50 | 2.1 | 422.8 | 

Especially when looking at the lower parallelism values, the total runtime gets progressively faster. On the other hand, the time of the API call to refresh a element increases possibly due to my Laptop not being fast enough to utilize all the parallelism or due to API rate limits. This results in very high parallelism even being a bit slower.

The same can also be seen with the SCC and SCI providers. With both having a larger jump from 1 to 10 and afterwards only small or no changes anymore.

And of course, the same will surely also be applicable to the BTP Provider (running 50 directory resources), right? Well, no. The BTP provider seems to have such extreme API limitations that with just a bit of parallelism, the time of the API calls increases dramatically. This increase is so large, in fact, that the total runtime never really improves, making parallelism for the BTP provider useless. 

| Parallelism | Time (s) | Refresh time per resource (ms) |
| ------------- | ---: | ---: |
| 1 | 8.4 | 149.2 | 
| 10 | 8.2 | 1367.9 | 
| 20 | 8.2 | 2501.6 | 
| 50 | 7.7 | 5255.0 | 

Another more general effect we can see is that the effect of parallelism seems to depend heavily on the time the APIs take. When using parallelism 10 for example the show below amounts of resources were refreshed at once. Showing that with slower performance of the APIs, more elements can be started simultaneously.
- **BTP**: 8.3 elements 
- **CF**: 4.5 elements
- **SCC**: 4.0 elementes
- **SCI**: 5.6 elements

_Calculated via: ( \<API Time per resource> * \<Number of resources = 50> ) / \<Time>_

---

## What is a small state?
Now let's get to the second question. How many resources are considered small for a state?

To answer this question, let's first change the initial setup a bit, as it is uncommon to have as many subaccounts as one has instances. The resulting setup has a total of 43 resources in the script, divided the following way:
- **BTP**: 1 Directory, 1 Subaccount, 4 Entitlement, 1 Environment, 2 Role, 3 Role Collection, 2 Service Instance, 3 Subscription, 1 Trust Configuration, 2 Generic Destination
- **CF**: 2 Space, 4 Service Instance
- **SCC**: 1 Subaccount, 2 Mapping, 6 Mapping Resource
- **SCI**: 3 Group Base, 5 User, 2 Application

To scale up the script, we will just be duplicating the resources inside the script by a factor f. Meaning f=1 has 43 elements, with f=5 having 215 elements.

### Results
The following is based on executing the script multiple times. To get this analysis done at some point, the amount of times executed is going down from 20 to 5 depending on the factor used. 

| Factor | Elements | Time (s) | Time per resource (s)* |
| ------------- | ------------- | ---: | ---: |
| 1 | 43 | 18.2 | 0.42 |
| 5 | 215 | 74.2 | 0.35 |
| 10 | 430 | 118.4 | 0.28 |
| 25 | 1075 | 223.9 | 0.21 |
| 50 | 2150 | 426.5 | 0.20 |

_* From here on out the time per resource is just calculcated via time/elements and not using the previous method of checking the resource resfresh/ API call time._

We already learned that entitlements are extremely slow, so lets also see how this behaves in a more complex scenario. In this case (using factor 1), removing them reduced the total time down to 9.4s. This means entitlements took around 50% of the time while only accounting for 10% of the resources. With factor 5 this even increases to a reduction of around 55%.

### Parallelism
We already know from the previous chapter that we cant expect any performance improvements from the BTP Provider alone, but how will it look in combinations with the other providers?

So let's check how much performance we can gain by increasing the parallelism using the factor 25 script from before. For this, each script was executed 3 times.

| Parallelism | Time (s) | Time per resource (s) 
| ------------- | ---: | ---: | 
| 1 | 2636.0 | 1.22 |
| 5 | 647.1 | 0.30 |
| 10 | 426.5 | 0.20 |
| 20 | 382.3 | 0.18 |
| 50 | 375.2 | 0.17 |

When comparing especially the 10 and 20 factors, one can see a difference of around 10%. When looking at the times that were possible before when executing the providers seperatly, one could see a difference of around 20% for the CF and SCI providers. CF and SCI provide around 40% of the resources. so the improvement dropping by around 50% (from 20% to 10%) is expected. This also means that the BTP provider had little improvmeent with increased parallelism, even when paired with different providers.

### Introducing Dependencies
Lastly, lets have a look on the effect of dependencies. For the start we will be ignoring the parallelism argument. 

Comparing the times with and without dependencies shows that there is no real difference between the two, the version with declared dependencies is even a bit faster in the runtime, as shown in the table below. This could either be coincidence or be due to the execution graph being faster to calculate.

| Factor | Time without dependencies (s) | Time with dependencies(s) |
| ------------- | ---: | ---: |
| 1 | 18.2 | 17.9 |
| 5 | 74.2 | 50.7 |
| 10 | 118.4 | 130.7 |
| 25 | 223.9 | 250.0 |
| 50 | 426.5 | 414.8 |

Now let's also introduce the parallelism again with the same logic as before (using factor 25).

| Parallelism | Time without dependencies (s) | Time with dependencies(s) |
| ------------- | ---: | ---: |
| 5 | 647.1 | 597.3 |
| 10 | 426.5 | 414.8 |
| 20 | 382.3 | 372.3 |
| 50 | 375.2 | 368.9 |

And similar to the results already shown without parallelism, the execution is slightly faster with dependencies included into the script. Additionaly, we can see again a 10% improvement between paralleism 10 and 20. 

### Dont forget the init
As a short sidenote at the end. When talking about the size of the state one should also keep in mind that small states can be a problem for the performance.

In general, having smaller states results in more states being required. This means that more `terraform init` need to be performed. On average the initialisation takes between 2 and 4 seconds per provider. Modules, especially remote ones, may also need around 2 seconds to initialise.

So, using these numbers, when one would use all four providers at once without any (remote) modules, it would take around 12 seconds to execute the init. So if by splitting the state, less than this time is saved, splitting becomes counterproductive.

Also, keep in mind that having very small states will diminish the effect seen before, where bigger state files had, to some extent, a better performance per resource than smaller state files.

---

## Summary
Now, let’s recap what can be learned from all these numbers:
1. The BTP provider is quite slow, which is especially the case for entitlements. Therefore, one should try to avoid resources from the provider in frequently changing state files.
2. When needing a lot of service instances, try to use the CF instances over the BTP instances if possible, due to their performance benefits.
3. The effect of parallelism is diminished when the BTP provider is involved. 
4. Don't forget to take the init into account when trying to improve plan performance by decreasing state size.
5. Similary, remember that the performance per resource improves the more resources a state has.

Lastly, when you want to calculate the expected runtime of your scripts, the following table can be used as a reference point for the maximum expected runtime per resource in the state:

| Provider | Time per resource (ms) | parallelism 10 adjusted |
| ------------- | ---: | ---: |
| **BTP** | 5000 | ∼600 |
| **CF** | 200 | ∼45 |
| **SCC** | 250 | ∼60 |
| **SCI** | 300 | ∼55 |

---

Thank you for reading this blog. I hope this analysis provided useful insights and practical takeaways. If you have feedback or questions, I would be happy to hear from you.

The data for the over 19 hours of runtime can be found in a [GitHub repo](https://github.com/christopherHJohn/performance-analysis-for-sap-terraform). So have a look there if you are interested in numbers.