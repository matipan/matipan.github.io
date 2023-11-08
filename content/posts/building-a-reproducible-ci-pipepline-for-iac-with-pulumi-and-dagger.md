---
title: "Streamlining CI/CD with Dagger: Beyond YAML"
date: 2023-11-07T00:00:00-03:00
tags: ["pulumi", "go", "aws", "dagger", "ci/cd"]
categories: ["infrastructure"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
disableHLJS: false
disableShare: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
UseHugoToc: true
cover:
    alt: "Building a reproducible CI pipeline for IaC with Pulumi and Dagger"
    relative: false
    hidden: true
editPost:
    URL: "https://github.com/matipan/matipan.github.io/tree/main/content"
    Text: "Suggest Changes"
    appendFilePath: true
---

[Dagger](https://dagger.io/) is a new tool that promises to fix the yaml and custom scripts mess that CI/CD currently is by building pipelines as code with one of the [supported SDKs](https://docs.dagger.io/). I'm in the process of learning this tool, understanding where it may fall short and where it shines and I decided that sharing some of the exploration I do and the learnings it leaves me would be useful.

**NOTE**: Dagger is in very active development so by the time you read this blog post it might already be deprecated.

## Background
We have a repository that holds all of our infrastructure in a declarative way using [Pulumi's](https://www.pulumi.com/) Go SDK. Every time we want to provision, change or delete some infrastructure we need to:
1. Create a pull request with the necessary changes
2. Wait for CI to show what changes will be applied by posting a comment to the PR
3. Merge it
4. Wait for CI on the main branch to apply those changes to the given environment

This process will utilize two operations pulumi provides:
* `preview`: show a diff of the changes that _would_ be applied.
* `up`: apply the requested changes.

We are going to build this workflow using Dagger and Pulumi's CLI and show how this differentiates from a traditional pipeline built with a CI specific tool such as Github actions. I will explore both the ["traditional" dagger client](#using-the-dagger-client) as well as the use of [dagger modules](#using-dagger-modules).
## Using the Dagger client
As was mentioned previously our CI process will have two separate workflows for:
* **Previewing changes**: used when a PR is created against `main` to execute `pulumi preview` and show the diff as a comment on the pull request.
* **Applying changes**: used when a commit is pushed to `main` to run `pulumi up` and apply the desired changes.

Since our infrastructure is built with Go we will use Dagger's Go SDK as well. To start building this pipeline we will add a `ci` folder to the root folder of our project with a `main.go` that instantiates a Dagger client:
```go
package main

import (
	"context"

	"dagger.io/dagger"
)
func main() {
	ctx := context.Background()
	client, err := dagger.Connect(ctx)
	if err != nil {
		log.Fatal(err)
	}

    // ...
```

The Dagger client is used to interact with the Dagger engine. It provides a code-generated SDK that performs GraphQL queries that in the end perform operations inside containers by leveraging [bulidkit](https://github.com/moby/buildkit). You could use this SDK to replace Dockerfiles completely but today we'll focus on using it to execute the `pulumi` CLI in a containerized environment.

Now, given that we need to support two operations (and both need the same flag) we need this program to manage arguments and execute different code based on those:
```go
// ...
var stack = flag.Parse("stack", "prod", "pulumi stack to use")

func main() {
	flag.Parse()

	// ...

	if len(os.Args) == 0 {
		log.Fatal("specify an operation. Possible values: preview,up")
	}

	switch os.Args[1] {
	case "preview":
		// Run `pulumi preview --stack <stack>`
	case "up":
		// Run `pulumi up --stack <stack>`
	}
}

```

Since with Dagger is containers all the way we can re-use existing images that have the dependencies we need. In this case all we need is pulumi, so we'll use the [official image](https://hub.docker.com/r/pulumi/pulumi) to build a `Container` that has the necessary go dependencies, code and secrets to run:
```go
func baseContainer(client *dagger.Client) *dagger.Container {
	pulumiToken := client.SetSecret("pulumi_token", os.Getenv("PULUMI_ACCESS_TOKEN"))
	awsAccessKey := client.SetSecret("aws_access_key", os.Getenv("AWS_ACCESS_KEY_ID"))
	awsSecretKey := client.SetSecret("aws_secret_key", os.Getenv("AWS_SECRET_ACCESS_KEY"))

	return client.
		Container().
		From("pulumi/pulumi:latest").
		WithSecretVariable("PULUMI_ACCESS_TOKEN", pulumiToken).
		WithSecretVariable("AWS_ACCESS_KEY_ID", awsAccessKey).
		WithSecretVariable("AWS_SECRET_ACCESS_KEY", awsSecretKey).
		WithMountedDirectory("/infra", client.Host().Directory(".")).
		WithWorkdir("/infra").
		WithEntrypoint([]string{"/bin/bash"}).
		WithExec([]string{"-c", "go mod tidy"})
}

```

In this `baseContainer` function we are building and returning a container that:
* Starts from pulumi's oficial image [pulumi/pulumi:latest](https://hub.docker.com/r/pulumi/pulumi)
* Has the credentials necessary to communicate with Pulumi and AWS obtained from the environment but stored in a secure way using [Dagger secrets](https://docs.dagger.io/723462/use-secrets/).
* Has the infrastructure code mounted in the working directory `/infra` and
* has all the go dependencies that our pulumi code requires.

This container can now be used to execute pulumi commands. To run the preview operation we can:
```go
// ...
func main() {
	// ...
	switch os.Args[1] {
	case "preview":
		out, err := baseContainer(client).
			WithExec([]string{"-c", fmt.Sprintf("pulumi preview --stack %s --non-interactive --diff", *stack)}).
			Stdout(ctx)
		if err != nil {
			log.Fatal(err)
		}

		fmt.Println(out)

	case "up":
		// Run `pulumi up`
	}
}
```

We are now ready to try it out! To run Dagger you can either: i) use Dagger's CLI to get a pretty output of what dagger is doing exactly (`dagger run go run ./ci preview --stack prod`); ii) or run go directly and get a silent output (`go run ./ci preview --stack prod`). Running this locally using Dagger's CLI shows a lot of output, so I'll show here only the relevant pulumi bits:
```
┃ @ Previewing update.....
┃ @ Previewing update.....
┃   pulumi:pulumi:Stack: (same)
┃     [urn=urn:pulumi:prod::networking::pulumi:pulumi:Stack::networking-prod]
┃         ~ aws:ecs/service:Service: (update)
┃             [id=arn:aws:ecs:us-west-2:831445021348:service/services-0187c44/ninjas]
┃             [urn=urn:pulumi:prod::networking::awsx:ecs:FargateService$aws:ecs/service:Service::ninjas]
┃             [provider=urn:pulumi:prod::networking::pulumi:providers:aws::default_5_35_0::321a9293-c7af-42be-a9fd-061afc82e317]
┃           ~ networkConfiguration: {
┃               ~ securityGroups: [
┃                   ~ [0]: "sg-0b2ee5e5403c1ca86" => "sg-0cf0cbf24b1e39535"
┃                 ]
┃             }
┃     - aws:ec2/securityGroup:SecurityGroup: (delete)
┃         [id=sg-0b2ee5e5403c1ca86]
┃         [urn=urn:pulumi:prod::networking::aws:ec2/securityGroup:SecurityGroup::svc-sg]
┃         description        : "Allow TCP traffic on port 8080 from the VPC"
┃         egress             : [
┃             [0]: {
┃                 cidrBlocks: [
┃                     [0]: "0.0.0.0/0"
┃                 ]
┃                 fromPort  : 0
┃                 protocol  : "-1"
┃                 self      : false
┃                 toPort    : 0
┃             }
┃         ]
┃         ingress            : [
┃             [0]: {
┃                 cidrBlocks : [
┃                     [0]: "10.1.0.0/16"
┃                 ]
┃                 description: "allow TCP traffic on 8080 from the VPC"
┃                 fromPort   : 8080
┃                 protocol   : "tcp"
┃                 self       : false
┃                 toPort     : 8080
┃             }
┃         ]
┃         name               : "svc-sg-6648e89"
┃         revokeRulesOnDelete: false
┃         vpcId              : "vpc-0ec0d8d2118fc95c5"
┃ Resources:
┃     ~ 1 to update
┃     - 1 to delete
┃     2 changes. 59 unchanged
```

So far we've run this locally, but we wanted this to be automated on a workflow. What changes do we need to make in order to adapt this to work in a CI environment? Nothing! We use Dagger so that we write code once and then run everywhere. The only CI-specific yaml we have yet to write is the workflow itself that offloads all of the heavy lifting to Dagger. Here is what a Github Actions workflow looks like for this preview functionality:
```yaml
name: 'preview'

on:
  pull_request:
    branches:
      - main

jobs:
  dagger:
    runs-on: ubuntu-latest
    permissions: write-all
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: '>=1.21'
      - name: Install Dagger CLI
        run: cd /usr/local && { curl -L https://dl.dagger.io/dagger/install.sh | sh; cd -; }
      - name: Preview infrastructure changes
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: "us-west-2"
        run: dagger run go run ./ci preview -stack=ninjastructure/prod
```

Other than "plugging" things together (env variables in this case) there is nothing special about this pipeline, all the important logic is hidden away inside our Dagger program.

Now we want to make the pipeline a bit more interesting. The diff that was calculated previously should be posted to the PR as a comment so that whoever is reviewing it can look at the PR comments and see what is being changed. This is where Dagger, being a code-first solution, really shines. We can extend our Go program using Github's SDK to communicate with it's API and post the comment with the output of the command. Lets change our `preview` code to include this:
```go
	// ...
	switch os.Args[1] {
	case "preview":
		out, err := baseContainer(client).
			WithExec([]string{"-c", fmt.Sprintf("pulumi preview --stack %s --non-interactive --diff", *stack)}).
			Stdout(ctx)
		if err != nil {
			log.Fatal(err)
		}

		// if the token is not specified we won't post anything
		token := os.Getenv("GITHUB_TOKEN")
		if token == "" {
			return
		}

		// GITHUB_REPOSITORY is in the format: `:owner/:repo`
		repo := strings.Split(os.Getenv("GITHUB_REPOSITORY"), "/")
		// On pull requests GITHUB_REF has the format: `refs/pull/:prNumber/merge`
		ref, err := strconv.Atoi(strings.Split(os.Getenv("GITHUB_REF"), "/")[2])
		if err != nil {
			log.Fatal(err)
		}
	
		return postComment(ctx, out, token, repo[0], repo[1], ref)
}

func postComment(ctx context.Context, content, githubToken, owner, repo string, pr int) error {
	body := fmt.Sprintf("```\n%s\n```", content)
	client := github.NewClient(nil).WithAuthToken(githubToken)
	_, _, err := client.Issues.CreateComment(ctx, owner, repo, pr, &github.IssueComment{
		Body: &body,
	})
	return err
}
```

If you want to test this changes locally you would have to provide github credentials. 

Here is a sample of this working at my repository:

![image that shows a comment in a Github PR containing a Pulumi diff that explains the infrastructure changes that want to be applied](/images/dagger-pr-comment.png)

I think it's interesting how dagger allows us to "tap" into the ecosystem of the language we are using in the same place where we are defining our CI process. While this example in particular is rather simple, imagine how this processes get complicated when you want to run more complicated process such as integration tests that require multiple containers running. I'm doing some exploration on this part that I hope to post soon, but in the meantime you can refer to [this YouTube video](https://www.youtube.com/watch?v=NjABt0owEeI) to see an example of a more complicated CI process being implemented with Dagger.

To finish off our CI, all we have to do is write the `up` operation using the `baseContainer` and specifying the `up` command:
```go
	case "up":
		out, err := baseContainer(client).
			WithExec([]string{"-c", fmt.Sprintf("pulumi up --stack %s --yes --skip-preview", *stack)}).
			Stdout(ctx)
		if err != nil {
			return err
		}
	
		fmt.Println(out)

```

And the `up` workflow looks very similar to the previous one:
```yaml
name: 'up'

on:
  push:
    branches:
      - main

jobs:
  dagger:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: '>=1.21'
      - name: Install Dagger CLI
        run: cd /usr/local && { curl -L https://dl.dagger.io/dagger/install.sh | sh; cd -; }
      - name: Preview infrastructure changes
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: "us-west-2"
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: dagger run go run ./ci up -stack=ninjastructure/prod
```

And we are done! We now have a pipeline that performs `preview` operations on PRs and leaves a comment showing the diff of what is going to be changed and an `up` operation on new commits in the main branch that will be apply changes to the infra directly. If you want to integrate this process with more tools, for example sending a message on Slack after changes have been applied you could use Slack's SDK or communicate with Slack's API directly using Go's standard library.
## Using Dagger modules

Using the Dagger client was quite straightforward, however, we had to manually build some form of "CLI". Parse arguments and flags and manage the logic of what the user is trying to call. The way I built it is not very "expressive" either, in order to understand what this CLI can do you would have to read the code. It wasn't that big of a deal given how simple the process is. But as this processes get more complicated (for example orchestrating apply commands on multiple environments based on specific conditions and files changed) the CLI will get more complicated and more parameterized and become less expressive overtime. 

The process we built is also not reusable. If somebody else is using pulumi as well and wants to apply this or a similar process they have to rebuild the whole thing from scratch. If you were building this using github actions, you would probably reuse an action that somebody else developed that has the functionality you need. So, how can we achieve this with Dagger? This is where [Dagger modules](https://daggerverse.dev) come in. Dagger modules allow us to encapsulate specific tasks (and even entire workflows) into functions that are written in one of the supported programming languages. The interesting bit is that Dagger then packages this functions into a GraphQL API that can be queried from any other language. This means that with modules you can essentially build a cross-language ecosystem of tasks and workflows that can be used by anybody. All you have to do is browse the [daggerverse](https://daggerverse.dev), find the module you are interested it and call those functions from your code base. This is all very exciting, now lets implement this same process using modules. We are still going to use Go's SDK because here we love Go. We first have to initialize the module, same as before we'll use the CI folder:
```sh
cd ci/
dagger mod init --sdk go --name pulumi
```

This will generate, among other stuff, a `dagger.json` file that contains the specification of the module:
```json
{
  "name": "infra",
  "sdk": "go"
}
```

In this same file we could potentially add dependencies. If somebody already developed a feature-rich `pulumi` module we could call `dagger mod use` and specify the URL, those dependencies would be added to `dagger.json` and from our code we call that module. Lets say the pulumi module did not support commenting on Github, well we could add the code to post those comments to our module using the output that the Pulumi module gave us. But, there is no pulumi module so we'll write everything from scratch. We can reuse everything that we built before with a small caveat: when using modules we do not use the dagger client as before. This is because Dagger generates code for us, this code contains the entire dagger SDK and any additional modules we referenced. So, lets refactor our pipeline to use this new generated code:
```go
package main

import (
	"context"
	"fmt"
	"strconv"
	"strings"

	"github.com/google/go-github/v56/github"
)

type Infra struct{}

func (m *Infra) Up(ctx context.Context, src *Directory, stack string, pulumiToken, awsAccessKey, awsSecretKey *Secret) (string, error) {
	return container(src, pulumiToken, awsAccessKey, awsSecretKey).
		WithExec([]string{"-c", fmt.Sprintf("pulumi preview --stack %s --non-interactive --diff", stack)}).
		Stdout(ctx)
}

func (m *Infra) Preview(ctx context.Context, src *Directory, stack string, pulumiToken, awsAccessKey, awsSecretKey *Secret, githubToken Optional[*Secret], repository, ghRef Optional[string]) (string, error) {
	out, err := container(src, pulumiToken, awsAccessKey, awsSecretKey).
		WithExec([]string{"-c", fmt.Sprintf("pulumi preview --stack %s --non-interactive --diff", stack)}).
		Stdout(ctx)
	if err != nil {
		return "", err
	}

	token, tokenSet := githubToken.Get()
	repoValue, repoSet := repository.Get()
	refValue, refSet := ghRef.Get()
	if !tokenSet || !repoSet || !refSet {
		return out, nil
	}

	// repository is in the format: `:owner/:repo`
	repo := strings.Split(repoValue, "/")
	// ref has the format `refs/pull/:prNumber/merge`
	ref, err := strconv.Atoi(strings.Split(refValue, "/")[2])
	if err != nil {
		return out, err
	}

	return out, postGithubComment(ctx, token, out, repo[0], repo[1], ref)
}

func postGithubComment(ctx context.Context, token *Secret, content, owner, repo string, pr int) error {
	githubToken, err := token.Plaintext(ctx)
	if err != nil {
		return err
	}

	body := fmt.Sprintf("```\n%s\n```", content)
	client := github.NewClient(nil).WithAuthToken(githubToken)
	_, _, err = client.Issues.CreateComment(ctx, owner, repo, pr, &github.IssueComment{
		Body: &body,
	})
	return err
}

func container(src *Directory, pulumiToken, awsAccessKey, awsSecretKey *Secret) *Container {
	return dag.
		Container().
		From("pulumi/pulumi:latest").
		WithSecretVariable("PULUMI_ACCESS_TOKEN", pulumiToken).
		WithSecretVariable("AWS_ACCESS_KEY_ID", awsAccessKey).
		WithSecretVariable("AWS_SECRET_ACCESS_KEY", awsSecretKey).
		WithMountedDirectory("/infra", src).
		WithWorkdir("/infra").
		WithEntrypoint([]string{"/bin/bash"}).
		WithExec([]string{"-c", "go mod tidy"})
}
```

As you can see in the `container` function, we are using the `dag` variable that it is not really declared anywhere in this file. And there are not dependencies to Dagger either. The variable is part of `dagger.gen.go`:
```go
dagger generated code
```

Using dagger's tooling we can now easily understand what this API is capable of. We can use `dagger functions` to understand what operations it supports:
```sh
$ dagger -m ./ci functions
✔ dagger functions [0.00s]
┃ object name   function name   description   return type
┃ *Infra        up                            String!
┃ *Infra        preview                       String!
• Engine: 03801e21c26e (version v0.9.3)
⧗ 2.77s ✔ 23 ∅ 2
```

And what arguments a given operation requires:
```sh
$ dagger -m ./ci call preview --help
✔ dagger call preview [0.00s]
┃ Usage:
┃   dagger call preview [flags]
┃
┃ Flags:
┃       --aws-access-key Secret
┃       --aws-secret-key Secret
┃       --gh-ref string
┃       --github-token Secret
┃   -h, --help                    help for preview
┃       --pulumi-token Secret
┃       --repository string
┃       --src Directory
┃       --stack string
```

Now, anybody that joins this project can easily understand what our CI process does and test it locally. Now, you may have noticed that we now have a huge list of arguments compared to before. On our previous iteration we were relying on the host's environment variables for every single credential we required. This is how traditional CI processes work. However, Dagger functions run in a sandboxed environment. This means that everything you have in the host will not be automatically available in the execution context of the function. We have to explicitly provide the credentials as arguments now:
```sh
$ dagger -m ./ci call preview --src "." --stack "ninjastructure/prod" --repository $GITHUB_REPOSITORY --gh-ref $GITHUB_REF --github-token $GITHUB_TOKEN --pulumi-token $PULUMI_TOKEN --aws-access-key $AWS_ACCESS_KEY_ID --aws-secret-key $AWS_SECRET_ACCESS_KEY
✔ dagger call preview [1m47.6s]
┃ Previewing update (ninjastructure/prod)
┃
┃ View Live: https://app.pulumi.com/ninjastructure/networking/prod/previews/81831d41-5ed0-4476-8bfa-cfa9c79f5b4f
┃
┃ go: downloading github.com/hashicorp/hcl v1.0.0
┃
┃ go: downloading google.golang.org/genproto v0.0.0-20230726155614-23370e0ffb3e
┃
┃ @ Previewing update......
┃   pulumi:pulumi:Stack: (same)
┃     [urn=urn:pulumi:prod::networking::pulumi:pulumi:Stack::networking-prod]
┃         ~ aws:ecs/service:Service: (update)
┃             [id=arn:aws:ecs:us-west-2:831445021348:service/services-0187c44/ninjas]
┃             [urn=urn:pulumi:prod::networking::awsx:ecs:FargateService$aws:ecs/service:Service::ninjas]
┃             [provider=urn:pulumi:prod::networking::pulumi:providers:aws::default_5_35_0::321a9293-c7af-42be-a9fd-061afc82e317]
┃           ~ networkConfiguration: {
┃               ~ securityGroups: [
┃                   ~ [0]: "sg-0b2ee5e5403c1ca86" => "sg-0cf0cbf24b1e39535"
┃                 ]
┃             }
┃     - aws:ec2/securityGroup:SecurityGroup: (delete)
┃         [id=sg-0b2ee5e5403c1ca86]
┃         [urn=urn:pulumi:prod::networking::aws:ec2/securityGroup:SecurityGroup::svc-sg]
┃         description        : "Allow TCP traffic on port 8080 from the VPC"
┃         egress             : [
┃             [0]: {
┃                 cidrBlocks: [
┃                     [0]: "0.0.0.0/0"
┃                 ]
┃                 fromPort  : 0
┃                 protocol  : "-1"
┃                 self      : false
┃                 toPort    : 0
┃             }
┃         ]
┃         ingress            : [
┃             [0]: {
┃                 cidrBlocks : [
┃                     [0]: "10.1.0.0/16"
┃                 ]
┃                 description: "allow TCP traffic on 8080 from the VPC"
┃                 fromPort   : 8080
┃                 protocol   : "tcp"
┃                 self       : false
┃                 toPort     : 8080
┃             }
┃         ]
┃         name               : "svc-sg-6648e89"
┃         revokeRulesOnDelete: false
┃         vpcId              : "vpc-0ec0d8d2118fc95c5"
┃ Resources:
┃     ~ 1 to update
┃     - 1 to delete
┃     2 changes. 59 unchanged
• Engine: 03801e21c26e (version v0.9.3)
⧗ 1m50.1s ✔ 64 ∅ 8
```

Running this in the CI process requires us to modify the github workflow and execute this exact same command specifying the credentials and environment variables available on the CI runner.

I believe this is a more expressive way of building CI systems. In the next section we will build this using native github actions and do a comparison of Dagger vs traditional CIs.
## Comparing Dagger to traditional CI solutions

To have a better understanding of whether Dagger gave us a better experience and how it conceptually differentiates from traditional CIs we have to at least implement this process using github actions. We could have leveraged a github action implemented by the Pulumi team to do this preview/up process with github comments included:
```yaml
name: Pulumi
on:
  - pull_request
jobs:
  preview:
    name: Preview
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-go@v3
        with:
          go-version: 'stable'
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-region: ${{ secrets.AWS_REGION }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      - run: go mod download
      - uses: pulumi/actions@v3
        with:
          command: preview
          stack-name: org-name/stack-name
          comment-on-pr: true
          github-token: ${{ secrets.GITHUB_TOKEN }}
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}
```

There are two main differences that I see with this implementation. First of all, this is not easily testable and extensible. We could leverage something like [arc]() or [gale]() to run this process locally, but when it comes to extending we would either have to build custom actions or do some setup magic and run custom scripts on top of this yaml. Again, this is a simple process, but CI/CD often gets more complicated as time goes on. And while right now this may not seem like an obvious problem, it always ends up becoming that critical yaml file that nobody wants to touch. The second main difference is more "conceptual" but it is important to point out because it requires a mindset shift when it comes to building CI. In the traditional pipeline, we can see a "statefull" approach, where each of the steps that are being executed are doing explicit things on the host (i.e the runner) that the subsequent action then reuses:
1. Checkout the code of the repository
2. Install the latest stable version of Go
3. Configure AWS's credentials (technically not needed, we could use env variables directly...)
4. Download go dependencies that pulumi's code uses.
5. Run pulumi preview

Every single one of those actions changed the underlying host in some way and left a state that the subsequent action would then use. This means that when you build a workflow you always operate directly on the host instead of "chaining" inputs/outputs like you do when building a dag, for example an Airflow dag. I don't particularly like this approach, it is one of the main things that makes this processed hard to test and more importantly very hard to understand.

With dagger modules we saw a different approach. When we were calling the function of the module, we had to explicitly provide all the required resources to perform the `preview` or `up` operation. This is because, as we mentioned previously, dagger functions run in a sandboxed environment that could even be executed on a different host! This, in my opinion, opens up interesting possibilities for building CI pipelines using reusable APIs that are deployed once and used everywhere. There might be occasions where this stateless approach falls short, but this new approach is only getting started. I think there is a lot of similarity with how the data world works. When you build data pipelines using tools like Airflow, the worker where the operator code gets executed usually only performs HTTP calls. For example, if you build a data pipeline that performs some data transformation using spark on EMR and then ingests that data on a datasource like clickhouse, your Airflow DAG will probably look something like this:
1. Use EMR Operator to execute some spark code. Airflow's worker will call AWS's API, ask for a given script to be executed and then wait for completion.
2. Use an ECS task or similar to run code that reads the data from S3 and ingest it into clickhouse.

## Conclusion
Dagger offers a compelling alternative to yaml and custom scripts in CI/CD pipelines. By leveraging code, we gain expressiveness, reusability, and the ability to integrate with a wide range of tools and services. As Dagger matures, I'm excited to see how it will continue to evolve and improve CI/CD practices.
