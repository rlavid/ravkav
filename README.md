# Nuclio Function Automation for Python and Jupyter

This is the Python package for automatically generating and deploying [nuclio](https://github.com/nuclio/nuclio) serverless functions from code, archives or Jupyter notebooks.
It provides a powerful mechanism for automating code and function generation, simple debugging, lifecycle management, and native integration into data-science tools.

#### The package provides the following features

- Automatically convert code/files + dependencies (environment, packages configuration, data/files)<br> into nuclio function spec or archive
- Automatically build and deploy nuclio functions (code, spec, or archive) onto a cluster
- Provide native integration into [Jupyter](https://jupyter.org/) IDE (Menu and %magic commands)
- Handle function+spec versioning and archiving against an external object storage (s3, http/s, git or iguazio)

#### What is nuclio?

Nuclio is a high-performance serverless platform running over Docker or Kubernetes that automates the development, operation, and scaling of code written in multiple supported languages.
Nuclio functions can be triggered via HTTP, popular messaging/streaming protocols, scheduled events, and in batch.
Nuclio can also run in the cloud as a managed offering, or on any Kubernetes cluster (cloud, on-prem, or edge)<br>
[read more about nuclio ...](https://github.com/nuclio/nuclio)

Nuclio and this package are an integral part of [Iguazio Data Science Platform](https://www.iguazio.com/), you can see many end to end usage examples and notebooks in [iguazio tutorial](https://github.com/v3io/tutorials) repository.

#### How does it work?

nuclio takes code + [function spec](https://nuclio.io/docs/latest/reference/function-configuration/function-configuration-reference/) + optional file artifacts and automatically convert them to auto-scaling services over Kubernetes.
the artifacts can be provided as a YAML file (with embedded code), as Dockerfiles, or as archives (Git or Zip).
function spec allow you to [define everything](https://nuclio.io/docs/latest/reference/function-configuration/function-configuration-reference/) from CPU/Mem/GPU requirements, package dependencies, environment variables, secrets, shared volumes, API gateway config, and more.<br>

this package is trying to simplify the configuration and deployment through more abstract APIs and `%nuclio` magic commands which eventually build the code + spec artifacts in YAML or Archive formats (archives are best used when additional files need to be packaged or for version control)

the `%nuclio` magic commands are simple to use, but may be limited, if you want more programmability use the `build_file`, `deploy_file`, and `deploy_code` Python functions from your code or a notebook cell.

## Usage

- [Installing](#installing)
- [Creating and debugging functions inside a notebook using `%nuclio` magic](#creating-and-debugging-functions-using-nuclio-magic)
- [Nuclio code section markers](#nuclio-code-section-markers) (`# nuclio: <marker>`)

## Installing

    pip install  --upgrade nuclio-jupyter

Install in a Jupyter Notebook by running the following in a cell

```
# nuclio: ignore
!pip install --upgrade nuclio-jupyter
```

to access the library use `import nuclio`

## Creating and debugging functions using `%nuclio` magic

`%nuclio` magic commands and some comment notations (e.g. `# nuclio: ignore`) help us provide non-intrusive hints as to how we want to convert the notebook into a full function + spec.
cells which we do not plan to include in the final function (e.g. prints, plots, debug code, etc.) are prefixed with `# nuclio: ignore`
if we want settings such as environment variables and package installations to automatically appear in the fucntion spec we use the `env` or `cmd` commands and those will copy them self into the function spec.

> Note: if we want to ignore many cells at the beginning of the notebook (e.g.  data exploration and model training) we can use `# nuclio: start` at the first relevant code cell instead of marking all the cells above with `# nuclio: ignore`.

after we finish writing the code we can simulate the code with the built-in nuclio `context` object
(see: debugging functions) and when we are done we can use the `export` command to generate the function YAML/archive or use `deploy` to automatically deploy the function on a nuclio/kubernetes cluster.

we can use other commands like `show` to print out the generated function + spec, `config` to set various spec params (like cpu/mem/gpu requirements, triggers, etc.), and `mount` to auto-mount shared volumes into the function.<br>

for more details use the `%nuclio help` or `%nuclio help <command>`.

### Example:

Can see the following example for configuring resources, writing and testing code, deploying the function, and testing the final function.
note serverless functions have an entry point (`handler`) which is called by the run time engine and triggers.
the handler carry two objects, a `context` (run-time objects like logger) and `event` (the body and other attributes delivered by the client or trigger).

We start with, import `nucilo` package, this initialize the `%nuclio` magic commands and `context` object this section should not be copied to the function so we mark this cell with `# nuclio: ignore` (or we can use `# nuclio: start-code` or `# nuclio: end-code` to mark the code section in the notebook).


```python
# nuclio: ignore
import nuclio
```

#### Function spec/configuration

the following sections set an environment variable, install desired package, and set some special configuration (e.g. set the base docker image used for the function).
note the environment variables and packages will be deployed in the notebook AND in the function, we can specify that we are interested in having them only locally (`-l`) or in nuclio spec (`-c`).
we can use local environment variables in those commands with `${VAR_NAME}`, see `help` for details.
>note: `%` is used for single line commands and `%%` means the command apply to the entire cell.

```
%nuclio cmd pip install textblob
%nuclio env TO_LANG=fr
%nuclio config spec.build.baseImage = "python:3.6-jessie"
```

magic commands only accept constant values or local environment variables as parameters if you are interested in more flexibility use the `nuclio.build_file()` or `nuclio.deploy_file()` Python functions.

#### Function code

In the cell you'd like to become the handler, you can use one of two ways:

- create a `def handler(context, event)` function (the traditional nuclio way)
- or mark a cell with `%%nuclio handler` which means this cell is the handler function (the Jupyter way)

when using the 2nd approach we mark the return line using `# nuclio:return` at the end of it.

#### Function build or deploy

once we are done we use the `%nuclio deploy` command to build the function and run it on a real cluster, note the deploy command return a valid HTTP end-point which can be used to test/use our real function.

deploy the code as nuclio function `nlp` under project `ai`:

    %nuclio deploy -n nlp -p ai

we can use `%nuclio build` if we only want to generate the function code + spec or archive and/or upload/commit them to an external repository without running them on the cluster this can also be used for automated CI/CD, functions can be built and pushed to GIT and trigger a CI process which will only deploy the function after it passed tests.

if you would like to see the generated code and YAML configuration file before you deploy use `%nuclio show` command

for more flexibility use the `nuclio.build_file()` or `nuclio.deploy_file()` API calls, see the example below:

```python
# nuclio: ignore
# deploy the notebook code with extra configuration (env vars, config, etc.)
spec = nuclio.ConfigSpec(config={'spec.maxReplicas': 2}, env={'EXTRA_VAR': 'something'})
addr = nuclio.deploy_file(name='nlp',project='ai',verbose=True, spec=spec, tag='v1.1')

# invoke the generated function
resp = requests.get('http://' + addr)
print(resp.text)
```

> Note: Cells containing `# nuclio: ignore` comment will be omitted in the build process.

### Example Notebook:

![](assets/nb-example2.png)

visit [this link](https://github.com/nuclio/nuclio-jupyter/blob/master/docs/nlp-example.ipynb) to see the complete notebook
, or check out this [other example](https://github.com/nuclio/nuclio-jupyter/blob/master/docs/nuclio-example.ipynb)

The generated function spec for the above notebook will look like:

```yaml
apiVersion: nuclio.io/v1
kind: Function
metadata:
  name: nuclio-example
spec:
  build:
    baseImage: python:3.6-jessie
    commands:
    - pip install textblob
    noBaseImagesPull: true
  env:
  - name: TO_LANG
    value: fr
  handler: handler:handler
  runtime: python:3.6
```

<a id="nuclio-code-section-markers"></a>
## Nuclio code section markers (`# nuclio: ignore/code-start/code-end`)

A user may want to include code sections in the notebook which should not 
translate to function code, for that he can use the following code markers (comment lines):

- `# nuclio : ignore` - ignore the current cell when building the function
- `# nuclio : code-start` - ignore any lines prior to this line/cell
- `# nuclio : code-end` - ignore any cell from this cell (included) to the end 

