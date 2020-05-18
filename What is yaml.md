# What is `YAML`?

> *YAML(YAML Ain’t Markup Language) is a human friendly data serialization standard for all programming languages.*

The following explaination is an extract from: 
https://medium.com/@yanxiaodi/using-azure-devops-pipelines-to-publish-the-nuget-package-from-github-repo-fb58be4e9be8

— yaml.org

`YAML` is designed to be human-friendly and work well with modern programming languages for common everyday tasks. It is similar to `JSON`. Actually, you could treat `YAML` as the superset of `JSON`. Every `JSON` file is also a valid `YAML` file. But the difference is that they have different priorities. The foremost goal of `JSON` is simplicity and universality so it is easy to generate and parse in every modern programming language. But for `YAML`, the foremost design goal is to improve human readability. So `YAML` is a little bit more complex to generate and parse.

Imagine how we can describe a basic data structure? There are three basic but important primitives: mappings (hashes/dictionaries), sequences (arrays/lists) and scalars (strings,/numbers). We could describe the structures of `JSON` like this:

- A collection of name/value pairs. An *object* starts with `{` and ends with `}`. Each name is followed by `:` and the name/value pairs are separated by `,`.
- A list/array of values. An *array* begins with `[` and ends with `]`. Values are separated by `,`.
- A *value* can be a *string* in double quotes, or a *number*, or `true` or `false`or `null`, or an *object* or an *array*. These structures can be nested.

Let us see how it is in `YAML`. There are similarities between `YAML` and `JSON`. We will not cover all the details of `YAML` because Azure DevOps Pipelines does not support all features of `YAML`.

# name/value

`YAML` also contains a set of name/value pairs. You do not need to use `{` and `}`. The left of `:` is the name and the right of `:` is the value. For example:

```
name: myFirstPipeline
```

Note that the *string* in `YAML` does not need to be quoted. However, they can be.

The value can be a string or number, or `true` or `false` or `null`, or an *object*. `YAML` uses indentation to indicate nested objects. 2 space indentation is preferred but not required. For example:

```
variables:
  var1: value1
  var2: value2
```

# collections

`YAML` uses `[]` to indicate an array. For example:

```
sequence: [1, 2, 3]
```

Another way is to use `-`, as shown below:

```
sequence:
  - item1
  - item2
```

# multiple data types

`|` indicates there are multiple data types available for the keyword. For example, `job | templateReference` means either a job definition or a template reference are allowed.

# comments

`JSON` does not support comment but you can use `#` for comments in `YAML`.

# The structure of `YAML` for Pipelines

When we set up the pipelines in Azure DevOps, we use **Stages**, **Jobs** and **Tasks** to describe a CI/CD process. One pipeline might contain one or more stages, such as “Build the app” and “Run tests”, etc. Every stage consists of one or more jobs. Every job contains one or more tasks. Let us see the hierarchy of the `YAML` file for the pipeline:

![img](https://miro.medium.com/max/36/1*DVdQ7ebBvJ2EliIhk1Gmhg.png?q=20)

![img](https://miro.medium.com/max/329/1*DVdQ7ebBvJ2EliIhk1Gmhg.png)

The hierarchy of the `YAML` file for the pipeline

You do not need all these levels because sometimes the pipeline only contains a few jobs so just tailor the steps for your specific requirement.
