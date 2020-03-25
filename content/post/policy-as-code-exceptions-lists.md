+++
banner = "banners/placeholder.png"
date = "2020-02-26T13:47:08+02:00"
tags = ["Policy as code", "conftest","OPA", "Rego"]
title = "Creating Exceptions Lists for Conftest in Rego"
+++

For those who don't know what conftest is, conftest is an open-source utility that helps engineers write tests against structured configuration data. At the time of writing, conftest supports many formats such as YAML, JSON, Dockerfile, and HCL/HCL2 amongst others. This variety of formats allows teams to validate their configurations regardless of the platform they belong to before making changes to live systems.

Conftest relies on the Rego language from Open Policy Agent for writing the assertions, which is a high-level declarative language that lets users specify policy as code and offload their policy decision-making from their software. This fact makes it perfect for integrating conftest with your development pipelines, and give early feedback to your engineers as to what deviations with the policy there are.

One of the limitations that conftest has, however, is that it is not context-aware; this means if you want or need to make any exceptions, you have to skip running conftest against those exceptions.

Here is an example. Let's imagine that in our organisation we have two policies regarding Dockerfiles: 

1. All binaries need to come via authorised methods to prevent shadow-it - which means we do not allow using `apt`, `yum`, or similar
2. All Dockerfiles need to use our pre-approved base images

The above policies can be described in rego for conftest as:

{{<gist menendezjaume 6b29b20a6a2a66d2ef11453c6c114975>}}

Now, let's imagine that in one of our pipelines, there is the following Dockerfile coming through:

{{<gist menendezjaume c7f439c464bfb90da3e14bb595c02d32>}}

This Dockerfile, as you can see, it is running a Django web application.

If we were going to run conftest against our Django Dockerfile - which we are because, in this scenario, we have integrated it with our pipeline - we would get the following output:

```
$ conftest test -p . Dockerfile 
FAIL - Dockerfile - [base_image_not_whitelisted] The base image in use is not whitelisted - "ruby:2.5"
FAIL - Dockerfile - [blacklisted_commands] The Dockerfile is installing unauthorised third-party software - "apt-get update -qq && apt-get install -y nodejs postgresql-client"
--------------------------------------------------------------------------------
PASS: 0/2
WARN: 0/2
FAIL: 2/2
```

And since there are errors in the test of our policies, our engineers would be prevented from merging that PR. However, in this case, we do want to allow this container to go through and get deployed. To solve this, we have to either not run conftest - which is not always an option on some pipelines - or to add an exception to our policies.

At this point, we find ourselves having to create exceptions using a tool that, as I mentioned, it is not context-aware. This condition leaves us with the only option of having to use the content of the object of the exception - this is, the Dockerfile - as the trigger of the exception itself. To do this, we are going to hash the input (as conftest understands it) and save it in what we will call an `exceptions list`:

{{<gist menendezjaume 0189202cc9ceea2bc8247a339421d5c4>}}

As you can see, I have created two lists - one for each one of the rules in the main package. I have also created a function called `is_exception` that returns true or false, depending on whether the hash of the input is in the given list.

To consume the exceptions package from our main package, we need to update it as follows:

{{<gist menendezjaume 6e53d7f7bf0cbe8a662f380cae75ef0d>}}

Importing the package and calling the `is_exception` function.

To make it as user friendly as possible, I have also created an `exception_message`, and make it part of the output of every rule. This way, when a conftest failure occurs, and there is a business case for an exception, users can simply add the hash of the input into the exceptions list and make it part of their PR.

Now, let's run conftest once more:

```
$ conftest test -p . Dockerfile 
FAIL - Dockerfile - [base_image_not_whitelisted] The base image in use is not whitelisted - "ruby:2.5" (Exception code: 8f1b53862732a9fdf2c5725e7f509ecf2d6c98ff2518cafbc8adac73865c1f29)
FAIL - Dockerfile - [blacklisted_commands] The Dockerfile is installing unauthorised third-party software - "apt-get update -qq && apt-get install -y nodejs postgresql-client" (Exception code: 8f1b53862732a9fdf2c5725e7f509ecf2d6c98ff2518cafbc8adac73865c1f29)
--------------------------------------------------------------------------------
PASS: 0/2
WARN: 0/2
FAIL: 2/2
```

As you can see, we obtain the same failures, but this time, they are accompanied by the exception code. Assuming that our engineer sees that, and believes that it is reasonable to add an exception, they would add it to our exceptions lists, resulting in something like:

{{<gist menendezjaume 709b05c527b746699f1382de0b2b6035>}}

If we run conftest once again:

```
$ conftest test -p . Dockerfile 
FAIL - Dockerfile - [base_image_not_whitelisted] The base image in use is not whitelisted - "ruby:2.5" (Exception code: 8f1b53862732a9fdf2c5725e7f509ecf2d6c98ff2518cafbc8adac73865c1f29)
--------------------------------------------------------------------------------
PASS: 1/2
WARN: 0/2
FAIL: 1/2
```

We can see that there has been only one failure. If the exception was also applicable for the other rule, then the hash should also be added to the other exceptions list; otherwise, the violation should be remediated.

Finally, to get the most out of this exceptions process, the repository where you keep your exceptions must leverage code owners and protected branches, ensuring that any changes have to be approved by a member of the security team.
