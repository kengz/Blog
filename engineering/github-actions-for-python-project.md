---
description: 2020/04/19
---

# Github Actions for Python Project

In any substantial software engineering project these trifecta of CI \(continuous integration\) must always exist:

1. build
2. test
3. code quality/test coverage

For long I have always used [CircleCI](https://circleci.com/) to do CI for my projects. It has served me well, but it wasn't always perfect. Ever since Github Actions officially came out I've been wanting to try it; this weekend I finally did, and I immediately migrated to Github Actions with minimal effort.

## The pain points of CircleCI

First, a quick rundown of how CircleCI works for the three steps outlined above. For a project on Github, CircleCI:

1. pulls a prebuilt Docker image of the project for the CI job
2. updates dependencies as needed for the latest commit
3. runs the test command
4. uploads the test coverage result

> Note that code quality analysis is done outside of CircleCI, and I use [CodeClimate](https://codeclimate.com/) for static code analysis. Once the tests are finished, step 4 uploads the test coverage results from CircleCI to CodeClimate.

These standard steps don't change. However, the problem with CircleCI is how difficult and time-consuming these steps can be.

For step 1, a project like SLM Lab will have a large Docker image \(&gt; 3GB\), which will take time to build and push to Docker Hub. You also have to take care of OS-specific dependencies that often are missing and are difficult to install from a bare OS image. Just look at the OS-level dependencies needed for SLM Lab:

```bash
RUN apt-get update && \
    apt-get install -y build-essential \
    curl nano git wget zip libstdc++6 \
    python3-dev zlib1g-dev libjpeg-dev cmake swig python-pyglet python3-opengl libboost-all-dev libsdl2-dev libosmesa6-dev patchelf ffmpeg xvfb && \
    rm -rf /var/lib/apt/lists/*
```

This image has to be tested before being pushed and used by CI to run its jobs. In step 2, we have to also install the project dependencies. Python does not have a nice dependency distribution system like Node.js or Ruby, thus installing packages can be really painful and slow. We use Conda for dependency management. An important thing to do at this step is to cache the project dependencies, which was impossible to do before V2, and I was going to try out Github Actions before updating to CircleCI V2.

Step 3 can be slow because the machines available from the free tier of CircleCI are slower. It takes 20 minutes to run the entire CI pipeline on CircleCI, and as we will see later it takes 5-10 minutes on Github Actions.

Step 4 is painful because there was not such thing as community-built Actions to do things like uploading test coverage to CodeClimate. The result was a complicated block of YAML to do 2 simple things: run test and upload results:

```yaml
# a snippet from my old CircleCI file
      - run:	
          name: Run Python tests	
          command: |	
            curl -L https://codeclimate.com/downloads/test-reporter/test-reporter-latest-linux-amd64 > ./cc-test-reporter	
            chmod +x ./cc-test-reporter	
            ./cc-test-reporter before-build	
            conda activate lab	
            python setup.py test	
            ./cc-test-reporter after-build -p ~/SLM-Lab --exit-code $?	
      - store_test_results:	
          path: htmlcov
```

## Migrating to Github Actions

By the end of my quick experimentation with Github Actions, I already got a working CI pipeline that's much cleaner and faster than my existing CircleCI pipeline, so I just turned off CircleCI and the migration was completed.

What makes Github Actions good:

* Great design for composing workflows
* Fast machines with prebuilt common OS-dependencies
* Community-driven Github Actions

I spent a short time going through the [intro sections](https://help.github.com/en/actions) of its documentation and [its differences with CircleCI](https://help.github.com/en/actions/migrating-to-github-actions/migrating-from-circleci-to-github-actions) before jumping directly into writing a new CI pipeline. A few design details to note:

* the main component of the pipeline is Workflow, which is triggered by Github events and runs a number of jobs in separate containers.
* Workflows run asynchronously and cannot be chained; jobs can be chained by adding a `needs` keyword.
* the Ubuntu container [preinstalls many useful dependencies](https://github.com/actions/virtual-environments/blob/master/images/linux/Ubuntu1804-README.md), which means complex libraries like SLM Lab doesn't need a custom Docker image to build its own.

Now let's get to building the CI pipeline by replicating the 4 steps in the CircleCI pipeline we're replacing.

Step 1 of building and using a custom Docker image with these OS-dependencies is no longer necessary since we can use the Github ubuntu image directly.

So, we can jump directly to step 2 to install the Conda dependencies. There's a community-built [action to setup miniconda](https://github.com/goanpeca/setup-miniconda) that you can use immediately. Now my Github Actions CI file looks like this: 

```yaml
name: CI

on:
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Setup Conda dependencies
      uses: goanpeca/setup-miniconda@v1
      with:
         activate-environment: lab
         environment-file: environment.yml
         python-version: 3.7
         auto-activate-base: false
    - name: Conda info
      shell: bash -l {0}  # activate the conda environment
      run: |
        conda info
        conda list
```

Next, for step 3 and 4 we need to run the test upload the test coverage result. Again, we use an [action to upload to CodeClimate](https://github.com/paambaati/codeclimate-action). A small detail is the token required `CC_TEST_REPORTER_ID` can be stored in the repository secrets and made available - this is a really nice first-class integration with Github. Altogether, we add the following block:

```yaml
    - name: Run tests
      shell: bash -l {0}
      run: |
        python setup.py test
    - name: Codeclimate test coverage
      uses: marsam/codeclimate-action@master
      with:
        command: echo 'Uploading test coverage'
        reporter-id: ${{ secrets.CC_TEST_REPORTER_ID }}
```

Now, all of the CI steps have been replicated and the CI workflow runs really fast in about 6 minutes. This is already a huge improvement in speed and simplicity over CircleCI.

For bonus, I added a [flake8 annotation action](https://github.com/rbialon/flake8-annotations) which will run flake8 on the commit, then nicely annotate a pull request with the messages.

```yaml
    - name: Setup flake8 annotations
      uses: rbialon/flake8-annotations@v1
    - name: Lint with flake8
      shell: bash -l {0}
      run: |
        pip install flake8
        # exit-zero treats all errors as warnings.
        flake8 . --ignore=E501 --count --exit-zero --statistics
```

That's all for the Github CI pipeline! I saved the file to [.github/workflows/ci.yml](https://github.com/kengz/SLM-Lab/blob/master/.github/workflows/ci.yml) of my repo, a status badge to README.md, and pushed the commit. See the [CI build here](https://github.com/kengz/SLM-Lab/actions?query=workflow%3ACI).

Lastly, I removed my old CircleCI pipeline. So long, old friend. Github Actions is just first-class when used with Github, and that is certainly expected given Github is entering the CI/CD space.

