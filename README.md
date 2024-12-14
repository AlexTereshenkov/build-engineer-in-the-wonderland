# Build Engineer in the Wonderland

## Preface

> The source code of this book is available on [GitHub](https://github.com/AlexTereshenkov/build-engineer-in-the-wonderland).

Why having a sophisticated build system in the first place? Do I need it if I build my product in a single language? What about the IDE support? Is having something like Bazel really worth it? If you are asking these questions, that's great — you are in the right place!

The answer to the questions above is "it really depends". It depends on the complexity of your codebase, on how much deviation is there from the industry standards, on the number of engineers you can allocate to the adoption and maintenance of a build system. Things like that.

There are lots of reasons why adopting tooling of Pants, Bazel, or Buck may be great and people are likely to mention reproducibility, better isolation for tests (sandboxing) leading to higher testing quality, and build speed (thanks to remote caching and remote execution). What's not being mentioned often enough is the **surface area** for the interaction with the build process. Imagine having your project pipeline being just `pants fmt lint check test package ::`  or `bazel build //... && bazel test //...`. This means no matter what kind of things you do in the repo, what kind of tools you employ and however you build the code, you'll always have a consistent way to invoke a build.

Let me explain.

When setting up a developer infrastructure, you often start small. For example, if your project is in Python, this could be just `pytest tests/`  call or if it's a Go project, this could be a simple Shell script that calls `go test ./...` and a couple of other commands. At some point later, you need to separate between running tests and running static checks that start taking a lot of time, so you write a tiny Makefile to call `make format`, `make lint`, and `make test` separately. If you have modified some files you don't intend to format at all, you don't want to waste running the `format`  goal on them, so you declare some dependencies in your `Makefile` . 

Now your project has grown and in addition to Python tests you also have some Rust tests, special SQL and YAML tests (for which you need different tools that not every engineer in the company is expected to have installed). Now your `make`  targets involve running `curl`  fetching the tools from the Internet and handling all kinds of weird logic. You realize your `Makefile` is a mess, and you don't trust it any longer, so you make your `build`  goal be this `build: clean compile`. Now every intermediate step is deleted in the `clean` since you lost confidence in your build setup. Ouch!

All those invocations are getting impossible to reason about so you rethink your approach to builds and write a cli in Go to make it easier to invoke the build commands you need to hide away the `Makefile`  disaster. Some things couldn't be ported to Go, so for a few special cases developers are still expected to run `make special-tests`. Of course running `go-cli`  in CI and `make target` for the same operation locally yields slightly different results and troubleshooting is exhaustive. The classics!

Years pass by and with the size of the project growing, you realize that you can't run all the tests any longer purely based on the dependency graph changes as your integration tests are too slow, so you end up hacking some tiny library to build the dependency graph between your components based on Git changes and call it "smart selective testing" leading to developers being confused every second time they try to figure out why a particular test wasn't selected in a CI run. No one really understands how the build works any longer and everyone in the developer experience team is terrified by the Frankenstein this thing has become.

Sounds familiar? Welcome to the build engineer in the wonderland!

## Introduction

This little "book" is a journey into an imaginary world of build systems where it's all sunshine and roses. The builds take practically no time, there are no flaky tests, the CI is battle tested and stable, and it takes minutes to get one's code into production. For some engineering organizations it may never be possible to get there which could be due to a myriad of reasons — understaffed developer productivity team, a large legacy codebase, or even little faith in the worthiness of having a build system in the first place. For others, it's something they believe in deeply and embrace from the very first line of the code in the repository; they may not be there yet across all points (or will never be!), but they strive to get as far as they reasonably can. 

What's important to keep in mind, though, is that like with many other things — it's a spectrum! We all have started this journey from a certain point having only so much knowledge, expertise, and experience. You may find builds taking one hour to be totally acceptable whereas others can't release unless they are able to re-build the same commit producing bit-for-bit identical binaries. It also makes little sense to compare your company to others; the amount of resources at a disposal of a big tech is not comparable to your four people team serving hundreds of engineers. Many engineers working at large organizations that are known for their successful adoption of a build system such as Bazel have an advantage since they worked at FAANG and similar companies and have seen first hand what it could be like. It's arguably really hard to find genuinely new solutions to the problems you might face unless the solution is shared with you by someone else. Don't worry too much — you can always move in the spectrum as you learn about the best practices and approaches other teams take; it's a marathon, not a sprint!

We'll start by exploring the local development environment and IDE which is where the code is written, tested, and explored. On a submission of a patch to a remote version control system server, a CI build starts, making sure the changes to the source code are acceptable. The infrastructure of CI plays an important role on this journey and is one of the most complicated mixture of components such as cloud, virtualization, and observability. Later on, it's not only the source code you start caring about, but also the dependency graph and build optimizations. Once you get most things sorted, you might go extra mile and start caring about reproducibility and tighter hermeticity than what build systems provide by default. If you are vigilant, you speculate over security of the release process and authenticity of your binary artifacts.

The book tries to focus primarily on build systems related topics, but does occasionally refers to more generic best practices applicable in most engineering settings. Software development best practices are too broad to cover in the book and you may refer to other excellent resources such as [Code Complete](https://en.wikipedia.org/wiki/Code_Complete) or [AWS DevOps Guidance handbook](https://docs.aws.amazon.com/wellarchitected/latest/devops-guidance/development-lifecycle.html) to learn more and see how well your organization is doing based on various metrics such as local environment provisioning time, average time to resolve vulnerabilities, or build success rate and pipeline stability.

In a few chapters covering individual domains, the parts of the development and build infrastructure are described in a style of a spectrum where you can get started with something trivial and then move up the ladder to something more sophisticated. The "better" extreme of the spectrum is where every build engineer likely wishes to end up, and certain aspects of the build process are, indeed, very approachable. Most of us, however, will never get this far across the whole build process and can only wonder what's it's like up there. After all, that's why it's called the wonderland!

PS. It was very difficult to get all the best practices and ideas that most engineers would find appealing at once, in the same book, but there's already so much ready to be shared and bring value to others. I don't think I'll ever "finish" this book and instead I would like to extend it further continuously as I discover some other concepts that are worth covering in this book.

## Chapter 1.1 — Local development environment

The ultimate goal is that it should be trivial to reproduce a dev environment where one can code, test, and debug. The dev environment can be used to run your applications and tools needed for working with a codebase. It also helps with the continuous integration and testing. 


Local development environment is often associated with developer experience (DevX) which is commonly referred to the workflow a developer follows when writing and testing software. The local development process falls under the inner dev loop. The inner dev loop is where developer works on the code and tests it before submitting a patch for a build to take place in a CI server potentially resulting in an artifact release (which is commonly known as outer dev loop). You may be familiar with virtualized (a VirtualBox or a VMWare disk image) or containerized development environments (such as Docker image files or [dev containers](https://containers.dev/)) that can be run locally or remotely.

Using artifact-based build systems may imply that there's little need for build isolation thanks to their sandboxing and reasonable hermeticity, however, you may need to exercise care running builds locally on your host as you'd need to make sure your system changes do not leave your build environment in favorable or unfavorable state.

### Options

> A host machine with system configuration automation.

This gets impractical fairly quickly, but it is still a very common way to set up the dev environment this way as it's very approachable. Having a couple of Shell scripts or an Ansible playbook to install necessary system packages might help to restore the state should things break. Relying on host machine for development, however, often results in instability of tests as the host resources may be used unintentionally (e.g. having your build system read system-level headers or execute globally installed binaries), so some kind if isolation is going be crucial.

> A virtual machine image with all the software pre-installed and configured.

A virtual machine (VM) may often be a good choice for creating a controlled, isolated, and reproducible environment. This is often true for the cases when using a containerized virtualization is difficult such as when doing cross-platform development (macOS vs Linux) or relying on system packages that may require a particular operating system configuration that is not compatible with the host (x86 vs arm64) or when installing certain libraries (e.g. dealing with GPU libraries such as CUDA) on hosts is error prone. Popular options include VirtualBox, VMware, Hyper-V (Windows only), and Parallels (macOS only).

[QEMU](https://www.qemu.org/) can be used for setting up development environments that target a different hardware architecture such as arm64 or a different operating system such as QNX. Unless your dev environment targets Windows, you are likely to be better off with your developers having Linux or macOS as their host operating systems. This is because many build systems are either unsupported or provide only partial support for Windows. Modern Windows versions provide excellent Linux integration with [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) and with enough automation, a machine with an operating system from Microsoft can become a decent Linux like development environment.

VMs are also often used as complementary dev environments for integration and exploratory testing providing a more lightweight alternative to test rigs. Their usage requires following a few steps: 

- creating and managing VM image (allocating resources, configuring networking, installing software and applications, planning and taking snapshots) 
- setting up for development (shared directories and providing appropriate file access via mounting, port forwarding)
- automating VM setup (using provisioning tools such as Vagrant, Ansible, or Chef)

If the target environment is not that different from the host of the engineers, going with a more lightweight approach than having the whole operating system with its own kernel, libraries, and dependencies in a standalone image might be preferable. This often involves having some kind of a containerization technology.

> A container image with all the software pre-installed and configured.

A solution that lets using a container such as [Docker image](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/) or a [devcontainer](https://code.visualstudio.com/docs/devcontainers/containers) would make it possible to interact with the resources packaged in the container. For instance, you could run a Python program using remote Python interpreters of different versions, each installed in separate containers. 

[Podman](https://podman.io/) — part of the open-source `libpod` library developed by Red Hat — has gained attention for certain features that address Docker's limitations, especially in terms of security and compatibility. You may want to take a closer look especially if you are a user of Fedora and RHEL-based Linux distributions. Having a dev environment in a container is particularly helpful if you ship your artifacts in form of container images. Having all tests pass in a container that was started from the very same image that is going to be deployed in production is very powerful. 

If your dev environment targets macOS, you might want to take a look at [Tart](https://tart.run/) which is a virtualization toolset to build, run and manage macOS and Linux virtual machines on Apple Silicon. Having the [integration with Cirrus CI](https://tart.run/integrations/cirrus-cli/) would let you build local pipelines for testing and automation.

Depending on the situation, it may also be interesting to explore cloud-based development platforms also known as Cloud Development Environment (CDE) such as [Gitpod](https://www.gitpod.io/) or [GitHub Codespaces](https://github.com/features/codespaces) if the nature of development permits that.

> Operating system with declarative builds.

An operating system that enables declarative builds such as [NixOS](https://nixos.org/). Ideally, every system change (declared in configuration files) can be recorded and tracked so that you could see how your current system state differs from the "golden" state (similarly to how you run `git diff`, you should be able to see what changes have been made in the OS using `nix-diff`). With its declarative and reproducible approach, NixOS can complement containerized environments since container images built with Nix are often more predictable and reproducible across environments. Nix can be used to create container images directly which would let you build container images in a fully reproducible way. [Tweag](https://www.tweag.io/group/scalable-builds/) is known for their work on using [Nix](https://nix.dev/manual/nix/2.18/introduction) package manager with Bazel with great success — see [Bazel and Nix: A Migration Experience](https://www.tweag.io/blog/2022-12-15-bazel-nix-migration-experience/) and a [codelab](https://github.com/tweag/nix_bazel_codelab) for you to try running Bazel with Nix. There are a few other options that rely on Nix package manager (and not on the operating system) where Nix is used to create reproducible environments such as [flox.dev](https://flox.dev/) and [devenv](https://devenv.sh/) with some CI integrations with first-class Nix support such as [Hercules CI](https://hercules-ci.com/).

## Chapter 1.2 — Integrated Development Environment

IDE plays a key role by providing tools and features that streamline and enhance the programming process. Using native language build tools would always result in better user experience compared to attempting to use IDE in setting of a multi-language monorepo project. This may be easier to deal for languages with more primitive tooling and build process such as Python or TypeScript, but may ruin the development workflow for Android or iOS developers who've been using Android Studio and XCode, respectively. Inadequate IDE support for monorepo build systems can often be a reason not to migrate — it's really hard to sell hermeticity and build speed if the basic things like autocomplete and code suggestions along with source previews and refactoring tools are not available after migrating to Bazel from Maven.

When setting up developers with IDE to work in a monorepo environment, make sure that the IDE is configured to work with a subset of a repository with quick navigation, search, and code introspection and developers can run and debug tests interactively.

Unrelated to build systems, giving flexibility on source code representation is often appealing to developers. This may get particularly relevant when migrating from a polyrepo to a monorepo setting which often implies using a single code formatting style. This is still a bit of a science fiction, but, technically, source code might be stored in an abstract way and not as plain text. For instance, by storing the code as a [concrete syntax tree](https://en.wikipedia.org/wiki/Parse_tree), developers may choose how they want to represent this as plain text using their own rules. With this, the formatting rules are obsolete as everyone is free to render the code however they want.

Built-in support for more sophisticated code representation would also create opportunity for programmatic refactoring. For instance, instead of writing an ad hoc regular expression, a rich query language could be used straight from the IDE user interface to identify every class with more than 10 methods or function invocations that do not pass all parameters by keyword arguments. A code intelligence platforms such as [Sourcegraph](https://sourcegraph.com/) may be of assistance here as well as [CodeQL](https://codeql.github.com/) engine enabled in your IDE of choice.

## Chapter 2 — Dependency graph

The shape of the dependency graph of a large codebase often plays an important role contributing to the build time. In certain cases, it may be necessary to go beyond language specific best practices and established testing conventions when developing in a large monorepo setting, especially in a codebase with a very tightly coupled dependency graph. 

Lots of the large, legacy codebases have evolved without having to pay attention to the dependency graph shape as it's not something you would normally think about in the early stages of a project. However, over time, as the project grows, poorly shaped dependency graph is likely to lead to various problems such as inability to have fast, incremental builds (when a frequently edited package is a transitive dependency of all test suites), complete large-scale refactoring (moving code members around may cause violations in import borders and code visibility rules), and reason about individual source code change (updating a comment in a module leads to failures in unrelated performance tests).

However, some of the ideas outlined in this chapter may make little sense in some organizations or settings; for instance, you might not need to care about the dependency graph from the build time perspective if you can run all the tests and they reliably complete in, say, 15 minutes. However, if the source code of your software is validated using expensive or long-running tests, you might need to be very vigilant with regard to what tests are scheduled for run as part of your builds. You might have remote cache and remote execution set up for your project, but if for every build you have to re-run all tests (because of the changes in transitive first party dependencies), nothing will make your tests to complete any faster however you run them.

### Metrics

Having some kind of mechanism to be able to track the shape of the dependency graph would be very helpful. This is because over time, changes will be made to the sources and it would be useful to know how a change in a patch to be merged affects the graph and whether there will be more tests running after the change is merged.

This could be addressed in a few ways. There is a certain benefit to have this information presented in the early stages of the patch submission process so that the effect a suggested change would have on the dependency graph can be brought up naturally as part of the code review process. This makes it possible to prevent changes having an adverse effect on the graph to be merged, but may slow down the cycle time. Alternatively, regular audits of the dependency graph metrics might be done by a handful of engineers who have an intimate knowledge of the business domain and design architecture to amend any degradation of the graph post factum.

Spotify has shared a very comprehensive overview of their approach for monitoring the dependency graph changes, see [Driving Architectural Improvements with Dependency Metrics](https://docs.google.com/presentation/d/1McLw_yWbPuR1UqaoowHMsu5LskPJX7kWETkB-DkqNpo/edit?usp=drivesdk&resourcekey=0-sVMAbv967ww2kWvJuzyN5w). There are lots of useful metrics that could be collected and tracked over time such as average count of tests modules that are going to be run for each source module change or average number of dependencies and reverse dependencies for each module.

### Tooling

It would be helpful to have a program that could query the dependency graph of a codebase to understand how different parts of the codebase are interconnected. Here are some key functionalities that would be particularly helpful:

- visual representation of the dependency graph as a directed graph where nodes represent modules and edges represent dependencies
- search (e.g. find modules with more than N 3rd party dependencies or a package that is a transitive dependency for N test suites) and dependency path querying (find the path in the graph between two nodes to understand direct and transitive dependencies)
- change impact analysis (highlight the impact of changes by showing what parts of the codebase would be affected if a particular module is modified)
- cyclic dependencies identification (particularly useful when migrating from a build tool that tolerated cycles to a build tool that doesn't)
- graph version comparison (compare dependency graphs between the main and a feature branch or a commit to see how dependencies have evolved over time)

For inspiration, on the topic of interactive graph visualization and navigation, see [Skyscope](https://github.com/tweag/skyscope) and with regards to interactive graph querying, see [dg-query](https://github.com/AlexTereshenkov/dg-query).

As a stretch goal, the tooling would benefit from having these features:

- integration with popular IDEs to provide real-time dependency insights during coding (e.g. suggesting where to move a newly added test case to keep tests with similar dependency chains in the same build target)
- optimization suggestions (e.g. how to refactor or simplify dependencies to improve code maintainability, for instance, identifying dependency between modules that should be broken to create a new independent component)
- scenario modeling when a package starts or stops depending on another package — what happens with the associated tests?
- license compliance (e.g. confirm that source code of a particular artifact to be released doesn't transitively depend on any third-party dependencies with unsupported licensing requirements)

### Granularity

Major build systems operate on individual files — it doesn't get more granular than that. This means that adding a comment or updating a docstring would still invalidate any cached results of running tests that exercised that file recently. If your codebase has a central dependency that can't be split into individual targets (e.g. a single database schema file used by multiple standalone applications), it might be worth considering writing an extension for your build system to make it able to reason about the individual sections of a file.

Another use case justifying extending the build system would be having a large configuration file with every section being used only by one application — it would really help if changes in an individual section had triggered dependency chain changes only for the application it is connected to. For simpler configuration files, this kind of change detection might be relatively easy to implement, however, it might get impractical quite soon attempting to achieve the same for the source code, particularly for very dynamic languages such as Python where adding a new test case might have an impact on execution of other test cases. It is therefore usually advisable to keep the granularity at the file level and not go into the individual line changes.

### Best practices

Some of these guidelines may apply only to certain programming languages whereas others are universal. Following these practices will help keeping the dependency graph in good shape.

> Import code members from where they are declared.

In some programming languages such as Python and JavaScript, it is possible to import a code member from any module where this member is imported (some kind of "transitive import") leading to totally unneeded intermediate dependency.

> Separate code with and without dependencies.

It is very common for modules that are external to your project to depend only on a few members in your project/application; this could be a constant or a convenience function. Placing constants that do not depend on anything (a string literal, an enum, or a dataclass with no logic) in a separate module will increase the chances that other projects would not depend on your project's modules that have dependencies leading to other project modules (or modules external to your team's ownership).

> Keep central dependencies free of any code.

This is relevant only to a handful of languages such as PHP (with `autoload.php`) or Python where in a package, the `__init__.py`  file is a dependency for all modules in a package (as whenever someone imports `from foo.bar import baz` , all the code in `foo/bar/__init__.py`  is evaluated). 

Being able to import items straight from the `__init__.py` is very convenient to avoid typing the full path exposing the implementation details and is considered to be a common practice among Pythonistas. However, the code in the `__init__.py`  file of a package is executed on an import of any member of that package (this is how Python package imports work). Making changes to the `__init__.py` file will make all package files to be considered changed as they all depend on that particular file, so having any code (especially if it changes frequently!) in that file is highly undesired.

> Keep the number of dependencies in source modules limited

A module that directly depends on many other modules is more likely to be considered changed when any of its dependencies change. Most often, size of the file (in terms of lines of code) goes hand in hand with number of dependencies: the larger the file is, the more dependencies it has, but it's not necessarily the case. For instance, a file containing boilerplate code or generated code may be thousands of lines, but have no or only a few dependencies. Following the best practices of the code design are likely to help keeping the number of dependencies limited, e.g. placing only related functionality into a module, having classes with a single responsibility, keeping the methods small, and so on.

> Keep utility modules small and well-factored

Keep utility modules small and well-factored, and avoid naming them something generic as "utils" or "helpers" unless the scope is very limited because giving modules very non-specific names encourages using them as a dumping ground for unrelated code. Ideally, the module can be aligned with a single topic and provide a clear, simple abstraction around that topic without too much indirection and you can name them after that topic.

> Be aware of test fixtures usage and dependency chain

Testing frameworks provide mechanisms for defining test configurations and shared fixtures at a directory or project level such as `jest.config.js`  for Jest or `conftest.py`  for Pytest. These config files are used to share fixtures across multiple test files which is great for code reusability, but terrible for the dependency graph shape. In practice, all test modules implicitly depend on such files in the current directory (where the config files is stored) and any ancestor directories (i.e. `src/project/test1.py` depends on `src/conftest.py`). Therefore, it's best to avoid having such config files in the root of the tests directory (if you store them in a central location) as they will be picked by all the tests in the directory, recursively. Instead, define your fixtures as close as possible to where they are used.

> Keep test modules small

Typically, a smaller test module means fewer dependencies and therefore lower probability that the test module will be considered changed in an arbitrary patch submission. If your organization is in position to run tests in parallel using sophisticated sharding logic, having smaller modules will help with that. Similarly, since caching of the results of previous test executions is done per module, you would be able to avoid re-running tests that are known to have already been completed recently.

In case you have large test modules hosting lots of test cases, consider refactoring them into multiple individual modules. However, make sure to group tests based on the dependencies of individual test cases. Blindly splitting a module with 40 tests into 4 modules each containing 10 random test cases may not yield the optimal dependency relationships still resulting in unnecessary tests to be run.

> Perform dependency audit

Depending on the build system of your choice, some of the dependency relationships might be discovered automatically (via static code analysis), but for others some manual work might be required. For instance, some code in a test module might make a subprocess call to an executable that is provided by a first party program from another package or a third party requirement and this dependency needs to be reflected in some sort of build metadata. Some build systems would require declaring every dependency to be brought into a sandbox whereas others may be less strict so your tests may still pass, but may get triggered on bumping a version of that third party requirement due to the incomplete dependency model. This applies to programs accessing non-code resources where they need to be present either at build or runtime.

> Apply source code visibility rules

You may want to add visibility rules to your application or a project to make sure its dependencies do not extend beyond what the authors have agreed on. The rules will also help to prevent other projects from starting to depend on your code. The visibility rules enforced in the repository serve a particular purpose: they ensure an incidental dependency is not introduced and minimize the chances the code being referred to will not be present at runtime. 

In codebases with languages that have visibility rules built into them such as Java or Scala (via access modifiers), one would use the mechanisms that enable declaring visibility of classes outside of their packages. For other languages such as Python, you would have to rely on conventions and agreements made within your engineering organization, optionally, writing some custom linting tools to help you spot prohibited imports to keep the dependency graph sane. Ideally, you would want to go beyond what is offered by any programming language and instead operate on files (or build targets) rather than individual code elements.

Having the visibility rules declared might be crucial in a large codebase, particularly if it has multiple, often independent, projects or components. By using the visibility feature, you can:

- Hide implementation logic behind an interface via a stable API
- Enforce a clean interface between components and prevent cross-project dependencies
- Deprecate safely by first raising warnings on deprecated usages and then at some point forbidding the imports
- Refactor fearlessly knowing no one else directly depends on your implementation details

Many popular build systems such as Bazel (see [Visibility](https://bazel.build/concepts/visibility)) and Pants (see [Validating dependencies](https://www.pantsbuild.org/stable/docs/using-pants/validating-dependencies)) provide mechanisms to allow you to control access to certain parts of your codebase to specific modules or packages. When agreeing on the rules that you would like to apply to the modules in your codebase you might need to consult any design documents and generate some graphs to be able to reason about the components in your codebase and the relationships between them. Asking the following questions would also help:

- For sources, should there be any restrictions on who can depend on the code in a directory or a package and what code in a directory or a package can depend on?
- For tests, if having multiple test suite types, should there be any restrictions on what code can be shared between those and where would you want to place any test support code?
- For non-code resources, should there be any restrictions on where these can be stored and used?
- How much effort would it be to refactor the codebase to fix violating imports and enforce the visibility ruleset (in a legacy codebase)?

Having the boundaries established between your codebase components has a few other benefits:

- Code reuse and code extraction: in a codebase model consisting of one or more shared libraries and multiple applications, an application may not be allowed to depend on code from a sibling application. That code would need to be extracted and, optionally, refactored to make it more generic and reusable by other applications.
- Faster builds: preventing tests modules to depend on other test modules would prevent unnecessary cache invalidations that might happen when after adding a new test case in one module you actually cause running all the tests from another test module that happens to be a reverse dependency.
- Safer deployments: if your components are released and deployed independently, a component wouldn't accidentally depend (potentially transitively) on code that is not included in the final deployment artifact such as a binary executable or an container image.

> Have a mechanism to declare inactive dependencies

It is possible that in your dependency graph there are dependencies between modules that in certain context should be ignored. For instance, modifying a plain copyright file arguably should not trigger any integration tests (even though this could be a transitive dependency for all tests in the repository if the file is copied into some releasable artifacts). There's a concept of "dormant dependencies" in Bazel where it is possible to enable/disable dependencies between targets that are marked as available or unavailable for dependency resolution.

A different, but related concept — an idea of a stable API — might also play a vital role in the way modified build targets are identified. Imagine a dependency chain "client application" → "database writer" → "database engine" where a client application needs a function to dump some data into the database. It invokes the "write" function from the database writer module which in turn relies on database engine to actually do the hard disk data manipulation. Now, if the database engine has changed the way the data is compressed when writing on disk, from the client perspective, nothing has changed as it would work exactly the same since the the database writer mechanism (in this case, the "stable interface") hasn't changed.

From the dependency graph perspective, however, changes in the engine would invalidate any tests exercising logic in the client application due to the transitive dependency chain. Changes in the engine might still justify running all tests exercising the writer, though. Likewise, changes in the writer might also justify running all client tests that are concerned with writing the application data into the database. It's only propagation of "changes" transitively beyond database writer that is undesired. Deactivating the dependency chains around stable interfaces might have a huge impact on the build time as it's not uncommon to have a package that is a very central dependency and is modified frequently causing invalidation of most cached build actions.

## Chapter 3 — Local build

Running builds locally should be no different to the builds taking place in CI (this will be almost guaranteed if using containers and an artifact based build system with high level of hermeticity), however, this may be hard to get in practice as there may still be some local configuration sneaking into the build environment. This is rarely an issue if the builds are run in some kind of sandboxed / isolated environment (not on a developer's machine), but it depends a lot on how hermetic you expect your build to be. So ideally, it should never be the case that a build passes on CI but fails locally, and vice versa.

Depending on how caching of build artifacts is set up and whether you depend on remote execution (discussed later) or not, you may want to consider fetching remote cache locally regularly — if the remote cache is only for the latest main branch, it may often be feasible to sync the cache locally. This is to cover the use case when an engineer pulls the latest from the repository, makes a change, and then runs tests — if the recently added code that wasn't yet tested locally has already been tested in CI, the test results could be reused.

It's often useful to be able to confirm the build passes some quick sanity checks before submitting a build to CI. For instance, it may be inefficient to trigger a build in a CI agent just to find out some code is not formatted or a linter discovers an unused variable. Starting a CI build is not free and with a limited number of agents in a pool (depending on the computing resources at your disposal), this may result in longer waiting time for other builds. 

These kind of checks are often done as part of [pre-commit](https://pre-commit.com/) hooks and are trivial to set up; make sure to make pre-commit checks fast and relevant to most use cases allowing developers to choose what checks make sense for their workflows (e.g. a data scientist is unlikely to benefit from a JavaScript linter check). Using pre-commit hooks can also help to prevent any secrets and sensitive data to be pushed to a shared remote repository. In addition to the language specific checks, build metadata can be also be controlled — visit the Chapter 4.3 Code review which highlights some of the steps that can be taken to keep the codebase build metadata correct.

It is also very helpful to collect all invocations of the build tool, both in local environments (where product engineers code) and in CI agents. Enabling telemetry in CI builds is straightforward as you control the runtime environment. With the local environment, you'd need to provide for the main build tool some kind of thin wrapper which would have telemetry submission features built in. You could use an existing solution similar to [Sentry](https://sentry.io/) or [Fluentd](https://www.fluentd.org/), but this could also be a minimalistic Bash script that would submit some payload to a remote telemetry server before and/or after the primary build tool invocation. You would typically be interested in knowing:
* the runtime environment (hardware architecture, operating system name and version)
* the command and its command line arguments as well as duration of the command execution
* build related statistics such as local/remote cache hit rates and metadata on intermediate build steps (should you collect them)

You would not normally want to rely on engineers telling you about local build failures that are caused by things that engineering productivity / build systems team controls such as build system misconfiguration or unexpected drop in support of certain command line flags. Any errors that occur during a build tool invocation should be submitted to a remote telemetry server, optionally, with alerting set up to be able to react on issues proactively.

Once all the local sanity checks pass and, optionally, tests and any other relevant build operations, it's time to submit a CI build.

## Chapter 4.1 — CI

The topic of CI has a very large scope so we'll only cover some of the pieces that are relevant for a build engineer. For a broader overview, please refer to the [Continuous Integration book](https://martinfowler.com/books/duvall.html) and Martin Fowler's [Continuous Integration blog post](https://martinfowler.com/articles/continuousIntegration.html). 

## Chapter 4.2 — Infrastructure

Getting infrastructure right is always hard as it involves so many moving parts and it's challenging to gain a deep expertise in all of them. The key points to focus on are related to simplicity and codification. If you are able to use an existing CI provider such as GitHub Actions, Buildkite, CircleCI or GitLab CI, this is great as it shifts a ton of complexity from your team. Using an in house solution such as Jenkins may work depending on your scale and human resources and dedication. For what it's worth, there are plenty of organizations that are migrating from Jenkins to Buildkite or GitLab CI and practically none doing the opposite. You may also want to keep your options open and evaluate less known providers such as [Cirrus CI](https://cirrus-ci.org/) or [Semaphore CI](https://semaphoreci.com/).

Your CI infrastructure is most likely going to be deployed in a public cloud such as [AWS](https://aws.amazon.com/), [GCP](https://cloud.google.com/), or [Azure](https://azure.microsoft.com/en-us/). Depending on your budget and circumstances, it may be worth evaluating other cloud providers such as [UpCloud](https://upcloud.com/), [OVH](https://www.ovhcloud.com/en-gb/"), or [Hetzner](https://www.hetzner.com/) which are known to be used by various engineering organizations for their CI infrastructure deployment. Make sure to run a fair amount of experiments, performance, and stability testing before committing to an emerging cloud provider. If you want to experiment developing the CI infrastructure in a public cloud, but are worrying about the unexpected costs or have security concerns, consider using some tools that would let you prototype and test your CI workflows locally using some cloud emulator tooling such as [Localstack](https://www.localstack.cloud/) for AWS or [Azurite](https://github.com/Azure/Azurite) for Azure.

In case of infrastructure breakages that may require larger changes throughout various components, you should be able to redeploy your infrastructure from code and have amount of [ClickOps](https://en.wiktionary.org/wiki/ClickOps) kept to a minimum. Embracing the [Everything as Code](https://docs.aws.amazon.com/wellarchitected/latest/devops-guidance/everything-as-code.html) philosophy and [GitOps](https://about.gitlab.com/topics/gitops/) would make everyone feel a lot safer and more incentivized to experiment knowing it's trivial to restore the desired state from the code. Apart from the infrastructure, configuration of any auxiliary tooling or build support software should be expressed in code; this often includes observability and monitoring (alerts and dashboards).

## Chapter 4.3 — Code review

In a large monorepo setting, in addition to the code review guidelines that are specific to the software engineering best practices and chosen programming language paradigms, it may be helpful to have guidelines for reviewing changes made to the build metadata files as part of patch submissions. This is not an exhaustive list, but it provides steps that would ensure a better consistency and accuracy of your builds. Ideally, these should be automated, however, a semi-automated approach for some of more complex checks might also be a reasonable compromise.

For the review steps relevant to the dependency graph health, visit Best practices in Chapter 2: Dependency graph.

#### Build targets

- Watch for files that have been added, but there are no build targets that own those files. Unless you have agreed that it's possible to add arbitrary files that are not going to be part of a build, if the resource is checked in, then it should be a dependency of some build target e.g. a Python source file reading the contents of a file or a macOS app including a license file into the app bundle. It's possible, however, that files have been checked in, but are not yet consumed in code — this happens often as part of a feature development when code is submitted in small patches. This should not cause a build to fail — it's probably not a good idea to have a hard requirement on declaring build targets for all files added unless they are actually used in the code.
- Watch for use of recursive globs in any of the build targets which is generally discouraged. There may be situations when this is justified, e.g. there is a directory with lots of nested sub-directories and all the files should be treated as a single entity, so they could be owned by a single target. Changes to any of the files therefore should imply change of a target (and therefore all of its dependents). 

#### Dependencies

- There should be guidelines that can be consulted when adding a new third party dependency; see one for [Adding a dependency to your Python project](https://alextereshenkov.github.io/adding-python-dependency.html).
- Depending on how strict you are with dependencies, you may need to consult the source code to see if there are any cases of accessing system resources such as configuration files or binary executables (e.g. as part of a subprocess call), and add those resources as explicit dependencies of the relevant build targets. For instance, if a Go program makes a call to `jq`  to process some JSON data, you may see a passing build if the test execution environment happens to have `jq`  installed, but would result in build failures should the test be run elsewhere.
- Depending on the programming language used for a program under review, it might be necessary to evaluate carefully any first party or third party dependencies that may not be immediately visible by taking a quick look at the top of the file where the imports are usually declared. For instance, interpreted languages such as Python support dynamic import of modules that can take place anywhere in the body of a program and JavaScript testing frameworks may be picking up files with fixtures located at arbitrary places in the codebase. Again, the test exercising the program might be passing if that file happen to be brought into the test environment (e.g. by another module needed by the test as well), but would fail should the dependency chain be modified in the future.

#### Visibility rules

- Changes to visibility rules should be examined very closely. Every change should be justified (e.g. does this package really need to be visible/accessible to everyone? Can this package actually import from another package if they are going to be deployed independently?)
- When adding a new application or project, ensure adequate code visibility rules are added to prevent any import violations (such as apps importing from each other); this relies on existing visibility rulesets that would, for instance, prohibit having a test module depend on another test module (for the reasons that adding a new test case in one module should not cause all tests in another module to be executed).

## Chapter 4.4 — Build

### Environment

After a push to a remote version control server, a CI build starts. It would most likely start in an ephemeral, fresh container of some sort which is done for isolation purposes. Depending on your circumstances, you may also benefit from some kind of [persistent worker](https://bazel.build/remote/persistent), i.e. a machine/container where the results of previous recent builds (and in particular any build system cache that is stored only in memory) are kept as they would be lost upon environment termination.

A build environment would ideally have readily available most of the data it needs to perform the build. This includes:

- latest build cache (compilation, linting, and testing results and artifacts)
- latest repository cache (with third party dependencies such as Python wheels or Debian packages)

Going a step further, by analyzing the nature of the build to start, a certain type of environment may be created in beforehand that would have some auxiliary tooling, test fixtures, or software pre-installed and pre-configured (e.g. knowing that a data processing software build will take place, a container with Hadoop instance started might be used to avoid waiting for the environment setup).

It should also be possible to set up test environments with constraints that would apply to test cases individually, for instance, if you want to run a particular system test in a container that only has 1 GB of RAM available or in an environment with a non-standard file system.

It is not common, but occasionally a build system might be used to run arbitrary programs because invoking a program is a lot easier since the user does not have to set up the runtime environment themselves. If the program is guaranteed to be reproducible and is free of unexpected side effects, it might be useful to cache the input and output artifacts. For instance, if caching running a Go program to process data in a text file stored somewhere on disk, subsequent runs of the same source code on the same text file should not result in any work done and the final artifacts produced could be fetched from the cache instead.

When builds are taking place on multiple platforms (for instance, CI agents are Linux machines, but engineers do their work on macOS devices), coming up with a solution to be able to share cache across platforms might become critical to be able to take advantage of it. For instance, building a JAR file (Java) on Linux and macOS devices with the same toolchain (the compiler version and the build container) would result in two bit-for-bit different files even though one could copy the files between the platforms and run it anywhere (the guarantees of JVM).

### Parallelism

Running CI at scale is likely to represent a significant part in your overall cloud computing bill. Depending on your financial situation, this would likely be a dial you might want to tune. For instance, if you have to move extremely fast, a valid approach would be to start all individual test cases in parallel even though this would result in certain redundancy in test environment setup. If your budget is tighter, you may want to craft the sharding strategy carefully making sure you can co-run certain test cases that require the same (and potentially expensive to set up) fixtures in the same environment to maximize reuse.

The parallelism applies to other individual build steps which often involve steps like formatting, static type checking, and linting. Depending on how widely pre-commit checks are adopted throughout the organizations (i.e. how likely is your code to be correctly formatted and linted before it's sent to CI?), you may want to trigger each build step in parallel. This means, for instance, that unit tests or even some expensive integration tests might start at the same time as the check responsible to confirm that code is formatted. 

Should all the steps succeed, you will end up with a green build faster thanks to this very aggressive parallelism. However, if the code is not formatted (or linter discovers an unused import), you have just wasted some cloud computing resources setting up the test environment and maybe even running some tests. What's worse, since the files would be reformatted, cache for any tests completed in the most recent build might be invalidated resulting in the tests being re-run. If a particular static analysis check is hard to get right or it is not available on developer machines (e.g. it requires a proprietary software license available only to CI agents), it may be best to run this check first before sending requests for tests with maximum parallelism. 

With increased parallelism, test cases may all start running in parallel at the same time. With sufficient number of executors, the testing stage would be as long as the slowest test case, so it makes sense to keep your test cases small and fast to run. Depending on the programming language and testing framework, your parallelism may be limited to a file in which case you may want to avoid files with many tests keeping the test runtime among them somewhat equally distributed. A downside of having extreme parallelism is that there's little opportunity for sharing any intermediate build results between the test execution processes. 

Imagine that every test case needs to have access to a compiled protobuf file — having a hundred test cases would lead to a hundred compilations of the very same file as the test cases are run independently. Ideally, there should be a mechanism that would let those test case executions to tap into a shared pool of all build actions that are currently being run (if they are not yet available in any cache stores). Some kind of hybrid approach may also make sense when quicker operations are just faster to do as part of an individual test case execution, but for slower build steps, it might be worth querying the shared pool of actions and even pause to wait for a completion of a certain action if it's in progress and should complete rather soon.

### Observability

Apart from standard logging, it is also very useful to keep information about execution of every build step and its execution time and cache hit rates, in particular. Having this information stored and tracked for every build makes engineers much more confident making changes in CI or build system configuration. For instance, recording how much time it takes for a step where `npm`  packages are pulled to prepare the test environment would enable you to set up an alert for the build team if it takes twice as long after a product team bumped the `Node.js`  version. Similarly, a notable performance degradation after migrating artifact storage between cloud providers could be spotted easily from a Grafana dashboard. 

There's a balance to strike as you don't want your observability strategy to be too fine-grained as it would make reasoning about overall performance much harder unless you have a way to aggregate the individual build actions. For this, you would need to focus on critical build steps — for instance, in a large Python codebase with hundreds of dependencies (with some notoriously famous to be hard to get right such as machine learning frameworks), the performance of `pip`  resolving dependencies is crucial to track closely to avoid any degradations that might happen with the version upgrades. If testing involving browser automation takes most of your CI testing time, you would likely want to watch the performance of Selenium related build steps when upgrading to the next major version.

### Dependencies 

Based on the security restrictions, the way a product team engineer would work with third-party dependencies can vary a lot. For example, in a CI build it may be not allowed to download a random package from a public repository index with the firewall settings blocking CI agents to access them. In this case, a formal request would need to be made to the security team (that may consult other stakeholders) that would evaluate the package and whether it meets the necessary criteria to be accepted and hosted in the company binary repository manager to be used in software builds. [Typosquatting](https://snyk.io/blog/typosquatting-attacks/) package names and [supply chain attacks](https://snyk.io/blog/npm-security-preventing-supply-chain-attacks/) are becoming a lot more common so it might make a lot of sense to restrict access to public package repositories from CI agents. See [Adding a dependency to your Python project](https://alextereshenkov.github.io/adding-python-dependency.html) to learn more about possible criteria to consider.

In a more relaxed environment, it's possible to start depending on third-party packages immediately and trigger CI builds that would download any external dependencies from arbitrary sources. In this case, it may be helpful to save every third-party dependency that is being downloaded from external repositories into the binary repository manager so that the build can be reproduced at arbitrary point of time in the future. On a subsequent build, the build step would check whether the package is present in the binary repository first before attempting to download from a web resource which would make your build:

- potentially a bit faster (if the artifacts are stored closer to the machines where the builds take place)
- more stable if those external repositories are unable to provide access to those dependencies (if they are down or packages have been removed)
- more secure (if the version of a package you use has been compromised, your builds wouldn't be affected)
- more reproducible as keeping any package that has ever been used in a build would let you rebuild an old commit either for troubleshooting or legal audit purposes.


For certain build environments, it may even make sense to build all third-party dependencies in-house to have them readily available for the builds, particularly if they may be hosted across multiple repositories. For instance, for C++ dependencies that need to be built for multiple architectures, you may end up developing a standalone pipeline responsible for building all the packages using some tool like [Conan](https://conan.io/).

### Pipeline

To maximize developer velocity, there should be multiple pipelines for individual products, projects, or teams in the organization so that components may be validated and deployed independently (e.g. it should be possible to update how billing works in production without requiring to build and deploy the main product).

No-op builds and builds not involving any nodes of the dependency graph should be very fast. Keep in mind, however, that modifying a Markdown file however innocent looking this change may be, can actually mean modifying a build target that may trigger some actions such as rebuilding the documentation and uploading the docs files onto production. Make sure to model those build targets and their dependencies accordingly.

When re-running a build, the stages of CI should be cached, e.g. if it's a rerun of a build due to a particular flaky test, there's no need to run any static checks or run the tests have already passed (if the results of tests are cached). This may be not be possible if every build takes place on a merge commit (i.e. merging the most recent main into the feature branch) which, depending on the shape of your dependency graph, may be invalidating the cache.

## Chapter 4.5 — Testing

It is vital that tests results are cached and information about the source of test result is available after every build (back to the requirement to collect metrics on cache hit rates). It may be that cached tests results are invalidated very often which may be an indication of a very tight dependency graph when changing an arbitrary file leads transitively to a high number of tests. It may help to invest some time in identifying the problematic parts of the graph and refactoring the code.

Depending on the programming language, it may be important to run your tests against multiple interpreters or compilers. For instance, it can be crucial to ensure that code is compatible against all support Python versions whereas other languages such as Go provide certain [guarantees](https://go.dev/doc/go1compat) of compatibility within a major version.

Depending on your setup and complexity of test harness, you may decide to build all targets (and run all tests) for every CI build. However, in larger repositories, this may soon become impractical as running all tests may take just too long. Some kind of partial, or incremental, build where only those targets that depend, transitively, on the modified files are re-built. For testing, one would typically query the dependency graph to identify the transitive dependents of the change set and run test targets discovered. You may need to apply some additional logic to keep the testing selection sane as with a very tight dependency graph, random changes may cause just too many unrelated tests to be scheduled. Refer to Chapter 2 Dependency graph to learn more about declaring inactive dependencies.

Most often, developers would be running tests locally before submitting a patch for CI to build. If you would allow them to write to the shared remote cache, then by the time a CI build starts, the test results submitted from their laptop would be available in the remote cache. This might be a dangerous strategy, however, as it increases the chances of cache poisoning which happens most often when a target is built in a non-hermetic environment resulting in production of nondeterministic output which is later stored in the remote cache. Incomplete or malformed output artifacts may also sneak into the remote cache leading to errors in other builds that happen to fetch that cache object. It's a lot more common to only let CI builds to write to the remote cache, but keep in mind that this still doesn't completely rule out a chance of breaking the remote cache.

In certain contexts, you may want to verify stability of your tests. To do that, you can configure some tests to be run multiple times. Note that this is different from retries when failing tests (unstable, or flaky tests) are re-run -- in this case, it's the passing tests that are executed repeatedly. This is often done to catch any randomness that might be present in your tests that is otherwise tricky to spot when the test run once and pass in a CI build. This could be an order in which objects in collections are iterated, dependency on the results of the previous test run, or state of the system resources. It's not uncommon to configure performance and some integrations test to be re-run hundreds or even thousands times. As an extra precaution, you could also configure your unit tests to be run a couple of times.

## Chapter 5 — Reproducibility

The ability to reproduce the bit-for-bit identical artifacts from the same source code is very powerful, but also very hard to enable in practice. Just as with level of parallelism or strictness of static checks, it's a dial one can tune. Unless you are legally obliged to be able to reproduce identical artifacts from the same commit, you most likely would care about some reasonable level of reproducibility which is required to ensure high level of cache hit rates. For instance, it would not be possible to effectively cache results of compilation of a build target if the compiler location is declared via an absolute path and includes hostname of an ephemeral CI container (that would be different in every CI build). To keep track of how hermetic your builds are, you might want to regularly validate builds in a clean and isolated environment to be able to detect any non-hermetic behavior. This could be a good candidate for a nightly/weekly CI job. 

There are a few low hanging fruit to ensure better hermeticity that would lead to having fewer factors that can have an impact on the build output:

- building in a constrained environment that has a known configuration (such as a Docker container or a Nix based sandbox); this should cut access to any resources and binaries you might have on your host machine locally or in CI
- controlling environment variables accessible to the build process (disabling access to environment variables and passing them explicitly)
- if having dependency on compiled third-party dependencies (e.g. Python packages may have native code that would be built using local compilers accessible on the system path), those are best to build beforehand and host in an internal binary repository
- avoiding fetching dependencies over the network in ad hoc manner instead of declaring them properly so that they become explicit dependencies of a build target
- refactor build tools and scripts that may depend on the order of files from the file system (or in archive files), which may vary between builds

The [Reproducible builds project](https://reproducible-builds.org/) hosts excellent documentation on this topic and lists plenty of [tools](https://reproducible-builds.org/tools/) that could help making certain build steps reproducible (getting just some build actions to be reproducible would immediately result in better cache hits even if the rest of them are not completely reproducible). You can also find out more about how Nix package manager and NixOS [achieve reproducibility in build infrastructure](https://reproducible.nixos.org/).

However, it may also make sense to stay pragmatic and accept a certain level of non-hermeticity in your builds. This would be particularly true if your resources are limited. For instance, companies in certain mission-critical industries or industries with strict safety requirements may build not only their third-party dependencies in-house, but also pretty much everything else from scratch, such as building compilers from sources (e.g. [Clang](https://clang.llvm.org/get_started.html)), [building Android kernel and drivers](https://android.googlesource.com/kernel/build/+/refs/heads/main/kleaf/docs/impl.md) or custom embedded Linux operating system image with [Yocto project](https://docs.yoctoproject.org/brief-yoctoprojectqs/index.html). This most certainly would be an overkill for a majority of organizations.

## Chapter 6 — Security

### Deployments

In addition to security principles that are universal such as [principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege) and some best practices mentioned in earlier chapters (creating lockfile with transitive dependencies, limiting access to public package repositories in CI agents, and making sure to keep external dependencies in internal binary repository), you may also be interested in how to ensure that only the code that has been properly reviewed would be accepted by the deployment system.

After implementing version control with code review process, you can add a rule so that at least one approval is required before merging a change request and pushes to the main branch without a pull request are not allowed. The builds would take place in a CI pipeline that might have necessary validation gates such as enforcing a code signing policy where only authorized users can sign code before it's deployed to production — GPG/PGP signatures can be used to sign each commit or tag. This means that only developers with valid keys are allowed to sign commits that will be accepted in production. Once this is done, you can ensure that all binary artifacts have been built from trusted source code within the CI pipeline and then use tools like `sha256sum` to verify the integrity of binaries at deployment time. Sophisticated build systems can be extended with custom actions to sign produced binary artifacts using GPG.

Once the path to deployment has been secured, you can restrict access to production environment to ensure that no insider can push unapproved code manually. This may be done using SSH key-based access with restricted permissions (e.g., with no direct write access to deployment directories) or using tools like PAM (Pluggable Authentication Modules) to implement role-based access control (RBAC) if a deployment involves operator access. Ideally, though, only the CI/CD pipeline should have access to the production environment to deploy code, preventing any human access and direct modification. For this, you can employ tools for auditing access logs using `auditd` to detect any unauthorized access attempts. Deploying in immutable infrastructure such as if using containerization to ensure that all deployed binaries are built and packaged in a controlled CI environment enhances security as containers would be the only deployment artifacts.

As a final step, it may be worth looking into setting up some monitoring/alerting for integrity of the deployments. Some specialized file integrity monitoring (FIM) tools like Tripwire or AIDE (Advanced Intrusion Detection Environment) can be used to monitor changes to files, ensuring that no unauthorized binary is introduced into production. Furthermore, software like SELinux or AppArmor can be used to restrict the actions that binary executables can take to limit the impact of malicious code.
