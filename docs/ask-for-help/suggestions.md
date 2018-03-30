# Ask for help

When you're initially getting started with BOSH and later when you are using BOSH to manage your production systems you may want assistance.

## Who creates BOSH?

BOSH is a freely-available open-source project that belongs to the Cloud Foundry Foundation. It continues to have full-time contributors from foundation members such as Pivotal, IBM, and others; as well as contributions to core BOSH and the huge array of community projects surrounding BOSH.

## Is there documentation?

Oh yes there is documentation.

https://bosh.io/docs is a comprehensive suite of documentation for deploying a BOSH environment to various cloud infrastructures, using BOSH, and debugging systems.

You also have the Ultimate Guide to BOSH right in front of you. It has been written to be read in a linear, sequential manner so that you progressively learn new concepts and never feel overwhelmed. If you want to search for specific topics, press `f` or click on the "Search" field on the https://ultimateguidetobosh.com website.

## Join the Slack channel

Register at https://slack.cloudfoundry.org, login, and join the `#bosh` channel.

Dmitriy Kalanin (`@dkalinin`) is the product manager for the BOSH project, its core repositories, and the https://bosh.io website and its subsystems. He also has a range of personal Github projects relating to BOSH at https://github.com/cppforlife. His Github username should be a warning: be careful what you tattoo on your social media usernames.

There are many other BOSH Core and general community members who might be available and might be able to answer your questions.

The downside of problems and resolutions being discussed in Slack can be that the thread is lost those who did not read it at the time. Google searches also do not find Slack conversations and their answers.

## Github Issues

Nearly every repository in the BOSH ecosystem is on Github, so I'll use the phrase "Github Issues" to cover "creating issues/tickets on a source code repository".

I personally like to create Github Issues. I've created many issues where I'm then subsequently discovered the solution, have updated the issue with the fix, and closed the issue. Or I updated the issue with a plausible workaround, yet left the issue open in case Dmitriy or someone else has a better solution.

Creating Github Issues requires that you choose a project source repository into which to create the issue. Typically you will choose between:

* https://github.com/cloudfoundry/bosh-cli for questions relating to the `bosh` CLI and its subcommands
* https://github.com/cloudfoundry/bosh-deployment for questions relating to deploying or upgrading your BOSH environment, its BOSH director, and other collocated systems (UAA, Credhub)
* https://github.com/cloudfoundry/bosh for questions relating to the behaviour of a running BOSH environment, its BOSH director, or your running instances
* https://github.com/cloudfoundry/bosh-agent if you specifically know that the bug or problem relates to the BOSH agent running on each BOSH instance
* https://github.com/starkandwayne/ultimate-guide-to-bosh for issues or problems with this book

If you are having issues or questions about individual BOSH releases or deployment manifests, then I recommend you create issues with their own respective Github issues.

## Consulting and commercial assistance

Stark & Wayne is a boutique consultancy that offers you work with you for short and long term engagements to roll out BOSH, migrate existing systems, and deploy new systems. I founded Stark & Wayne in 2012 and it has grown from strength to strength with the rising tide of Cloud Foundry and the shared success of all Cloud Foundry Foundation members.

We would very much like the opportunity to help you on your journey with BOSH, CI/CD, and devops.
