## Code scanning

Code scanning enables developers to integrate security analysis tooling into their developing workflow. In this exercise, we will focus on the CodeQL static analysis tooling that helps developers detect common vulnerabilities and coding errors.

### Contents

- [Enabling code scanning](#enabling-code-scanning)
- [Reviewing any failed analysis job](#reviewing-any-failed-analysis-job)
- [Using context and expressions to modify build](#using-context-and-expressions-to-modify-build)
- [Reviewing and managing results](#reviewing-and-managing-results)
- [Triaging a result in a PR](#triaging-a-result-in-a-pr)
- [Customizing CodeQL configuration](#customizing-codeql-configuration)
- [Adding your own code scanning suite to exclude rules](#adding-your-own-code-scanning-suite-to-exclude-rules)

### _**Lab 3**_

#### Enabling code scanning

1. On the `Security` tab, in the **Vulnerability alerts** section, click **Code scanning**, and then click the **Configure scanning tool** button. You will be directed to the `Code security and analysis` section of the repository settings. Click the drop down button **Set up** which provides two choices (Default, Advanced) - please select **Advanced** for this lab in order to generate the CodeQL related workflow file.

3. Review the created Action workflow file `codeql-analysis.yml` and choose `Start commit` to accept the default proposed workflow.

4. Head over to the `Actions` tab to see the created workflow in action. Click on the workflow to view details and status for each analysis job.


#### Reviewing any failed analysis job

CodeQL requires a build of compiled languages. An analysis job can fail if our *autobuilder* is unable to build a program to extract an analysis database.

1. Inside the workflow you'll see a list of jobs on the left. Click on the Java job to view the logging output and review any errors to determine if there's a build failure.

2. The `autobuild` compilation failure appears to be caused by a JDK version mismatch. Our project targets JDK version 15. How can we check the Java version that the GitHub hosted runner is using? Does the logging output provide any helpful information?

    <details>
    <summary>Solution</summary>

    - GitHub saves workflow files in the `.github/workflows` directory of your repository. You can add a command to the existing `codeql-analysis.yml` workflow to output the Java version.  Add this anywhere before the step that is failing to help in your debugging:

    ```yaml
    - run: |
        echo "java version"
        java -version
    ```

    </details>

3. The previous debugging has concluded that we have a mismatch.  Resolve the JDK version issue by using the `setup-java` Action in `codeql-analysis.yml` to explicitly specify a version.  This should be added to the workflow before the `autobuild` step to properly configure the runtime environment before the build.

    <details>
    <summary>Solution</summary>

        uses: actions/setup-java@v3
        with:
            java-version: 16
            distribution: 'microsoft'

    </details>


#### Using context and expressions to modify build

How would you [modify](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions) the workflow such that the autobuild step only targets compiled languages (`java` in our repository)?

  <details>
  <summary>Solution</summary>

  You can run this step for only `Java` analysis when you use the `if` expression and `matrix` context.

  ```yaml
  - if: matrix.language == 'java'  
    uses: github/codeql-action/autobuild@v2
  ```
  </details>

#### Reviewing and managing results

1. On the `Security` tab, view the `Code scanning alerts`.

2. For a result, determine:
    1. The issue reported.
    1. The corresponding query id.
    1. Its `Common Weakness Enumeration` identifier.
    1. The recommendation to solve the issue.
    1. The path from the `source` to the `sink`. Where would you apply a fix?
    1. Is it a *true positive* or *false positive*?

#### Triaging a result in a PR

The default workflow configuration enables code scanning on PRs.
Follow the next steps to see it in action.

1. Add a vulnerable snippet of code and commit it to a patch branch and create a PR.

    Make the following change in `frontend/src/components/AuthorizationCallback.vue:27`

    ```javascript
     - if (this.hasCode && this.hasState) {
     + eval(this.code)    
     + if (this.hasCode && this.hasState) {
    ```
2. Is the vulnerability detected in your PR?

3. You can also configure the check failures for code scanning. Go into the `Code security and analysis` settings and modify the Check Failures. Set it to `Only critical/ Only errors` and see how that affects the code scanning status check for subsequent PR checks. In the next steps, you will be enabling additional query suites that have other severity types.

#### _Stretch Exercise 1: Fixing false positive results_

If you have identified a false positive, how would you deal with that? What if this is a common pattern within your applications?

#### _Stretch Exercise 2: Enabling code scanning on your own repository_

So far you've learned how to enable Dependabot, secret scanning, and code scanning. Try enabling this on your own repository, and see what kind of results you get!

#### Customizing CodeQL Configuration

By default, CodeQL uses a selection of queries that provide high quality security results.
However, you might want to change this behavior to:

- Include code-quality queries.
- Include queries with a lower signal to noise ratio to detect more potential issues.
- To exclude queries in the default pack because they generate *false positives* for your architecture.
- Include custom queries written for your project.

1.  Create the file `.github/codeql/codeql-config.yml` and enable the `security-and-quality` suite.

    **Hints**

    1. A configuration file contains a key `queries` where you can specify additional queries as follows

        ```yaml
        name: "My CodeQL config"

        queries:
            - uses: <insert your query suite>
        ```
2. Enable your custom configuration in the code scanning workflow file `.github/codeql/codeql-config.yml`

    **Hints**

    1. The `init` action supports a `config-file` parameter to specify a configuration file.

3. After the code scanning action has completed, are there new code scanning results?

#### Adding your own code scanning suite to exclude rules

The queries that are executed is determined by the code scanning suite for a target language.
You can create your own code scanning suite to change the set of included queries.

By creating our own [code scanning suite](https://codeql.github.com/docs/codeql-cli/creating-codeql-query-suites/), we can exclude the rule that caused the false positive in our Java project.

1. Create the file `custom-queries/code-scanning.qls` with the contents

    ```yaml
    # Reusing existing QL Pack
    - import: codeql-suites/javascript-code-scanning.qls
      from: codeql-javascript
    - import: codeql-suites/java-code-scanning.qls
      from: codeql-java
    - import: codeql-suites/python-code-scanning.qls
      from: codeql-python
    - import: codeql-suites/go-code-scanning.qls
      from: codeql-go
    - exclude:
        id:
        - <insert rule id of false positive>
    ```

2. Configure the file `.github/codeql/codeql-config.yml` to use our suite.

    **Hint**: We are now running both the default code scanning suite and our own custom suite.
    To prevent CodeQL from resolving queries twice, disable the default queries with the option `disable-default-queries: true`

<details>
<summary>Solution</summary>

```yaml
name: "My CodeQL config"

disable-default-queries: true

queries:
    - uses: ./custom-queries/code-scanning.qls
```
</details>

3. After the code scanning action has completed, is the false positive still there?

4. Try running additional queries with `security-extended` or `security-and-quality`. What kind of results do you see?

**Note**: If you want to use these additional query suites and the custom query suite you've made, make sure to import the proper query packs to continue to exclude certain queries.

<details>
<summary>Solution</summary>

```yaml
# Reusing existing QL Pack
- import: codeql-suites/javascript-security-and-quality.qls
  from: codeql-javascript
- import: codeql-suites/java-security-and-quality.qls
  from: codeql-java
- import: codeql-suites/python-security-and-quality.qls
  from: codeql-python
- import: codeql-suites/go-security-and-quality.qls
  from: codeql-go
- exclude:
  id:
    - java/spring-disabled-csrf-protection
```
</details>

5. Try specifying directories to scan or not to scan. Note that this is only supported for interpreted languages, such as javascript/typescript, python, ruby, etc. Why would you include this in the configuration?

<details>
<summary>Solution</summary>

```yaml
name: "My CodeQL config"

disable-default-queries: true

queries:
    - uses: ./custom-queries/code-scanning.qls

paths-ignore:
 - '**/test/**'
```
</details>

#### Understanding how to add a custom query

One of the strong suites of CodeQL is its high-level language QL that can be used to write your own queries.
_If you have experience with CodeQL and have come up with your own query so far, take this time to commit those changes and see if any alerts were produced._
Regardless of experience, the next steps show you how to add one.

1. Make sure to create a QL pack file. For example, `custom-queries/go/qlpack.yml` with the contents

    ```yaml
    name: my-go-queries
    version: 0.0.0
    libraryPathDependencies:
        - codeql-go
    ```

    This file creates a [QL query pack](https://help.semmle.com/codeql/codeql-cli/reference/qlpack-overview.html) used to organize query files and their dependencies.

2. Then, create the actual query file. For example, `custom-queries/go/jwt.ql` with the contents

    ```ql
    /**
    * @name Missing token verification
    * @description Missing token verification
    * @id go/jwt-sign-check
    * @kind problem
    * @problem.severity warning
    * @precision high
    * @tags security
    */
    import go
    /*
    * Identify processors that are missing the token verification:
    *
    * func(token *jwt.Token) (interface{}, error) {
    *    // Don't forget to validate the alg is what you expect:
    *    //if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
    *    //        return nil, fmt.Errorf("Unexpected signing method: %v", token.Header["alg"])
    *    //}
    *    ...
    * }
    */
    from FuncLit f
    where
        // Identify the function via the argument part of the its signature
        //     func(token *jwt.Token) (interface{}, error) { ... }
        f.getParameter(0).getType() instanceof PointerType and
        f.getParameter(0).getType().(PointerType).getBaseType().getName() = "Token" and
        f.getParameter(0).getType().(PointerType).getBaseType().getPackage().getName() = "jwt" and
        // and check whether it uses jwt.SigningMethodHMAC in any way
        not exists(TypeExpr t |
            f.getBody().getAChild*() = t and
            t.getType().getName() = "SigningMethodHMAC" and
            t.getType().getPackage().getName() = "jwt"
        )
    select f, "This function should be using jwt.SigningMethodHMAC"
    ```
3. Then, add the query to the CodeQL configuration file `.github/codeql/codeql-config.yml`

**Hint** The `uses` key accepts repository relative paths.

<details>
<summary>Solution</summary>

```yaml
name: "My CodeQL config"

disable-default-queries: true

queries:
    - uses: security-and-quality
    - uses: ./custom-queries/code-scanning.qls
    - uses: ./custom-queries/go/jwt.ql

```
</details>

#### _Stretch Exercise 3: Adding a custom query from an external repository_

How would you incorporate that query/queries from other repositories?

<details>
<summary>Solution</summary>

```yaml
name: "CodeQL Config"

disable-default-queries: false

queries:
  - name: go-custom-queries
    uses: {owner}/{repository}/<path-to-query>@<some-branch>
  - uses: security-and-quality
```
</details>

#### _Stretch Exercise 3a: Uploading the SARIF as a workflow artifact_

The output of the `github/codeql-action/analyze@v1` is a SARIF. You may want to obtain this when you want to look into the SARIF directly on your local machine and/or view it in SARIF viewer tool outside of GitHub. What action should we use to upload the SARIF as an artifact?
<details>
<summary>Solution</summary>

```yaml
    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v1
      with:
        output: code-scanning-results

    - name: Upload SARIF as a Build Artifact
      uses: actions/upload-artifact@v2
      with:
        name: sarif
        path: code-scanning-results
        retention-days: 7
```
</details>

#### _Stretch Exercise 3b: Uploading CodeQL databases as workflow artifacts_

By looking at the logs, where does CodeQL output the CodeQL databases, and similar to the previous exercise, how do we upload this? Furthermore, you'll be able to tell where the CodeQL binary lives as well, so you can pull the path to the CodeQL binary on the GitHub hosted runner into the Actions workflow.


**Hints**
- [How to set outputs of a step](https://github.com/actions/toolkit/blob/main/docs/commands.md)
- [CodeQL version in ubuntu-latest GitHub hosted runner](https://github.com/actions/virtual-environments/blob/main/images/linux/Ubuntu2004-README.md#tools)
- [CodeQL CLI Reference](https://codeql.github.com/docs/codeql-cli/manual/)
  - [codeql database bundle](https://codeql.github.com/docs/codeql-cli/manual/database-bundle/)

<details>
<summary>Solutions</summary>

```yaml
    - name: Upload CodeQL database
      id: codeql-database-bundle
      env:
        LANGUAGE: ${{ matrix.language }}
        CODEQL_PATH: /opt/hostedtoolcache/CodeQL/<codeql-bundle-name>/x64/codeql/codeql
      run: |
        CODEQL_DATABASE="/home/runner/work/_temp/codeql_databases/$LANGUAGE"
        CODEQL_ZIP_OUTPUT="codeql-database-$LANGUAGE.zip"

        $CODEQL_PATH database bundle $CODEQL_DATABASE --output=$CODEQL_ZIP_OUTPUT
        echo "::set-output name=zip::$CODEQL_ZIP_OUTPUT"

    - name: Upload CodeQL database
      uses: actions/upload-artifact@v2
      with:
        name: ${{ matrix.language }}-db
        path: ${{ steps.codeql-database-bundle.outputs.zip }}
```

The solution above shows how to use the CLI to zip a CodeQL database. GitHub hosted runners are regularly updated, so be aware of the CodeQL bundle version you're using.
Here's another way of uploading a CodeQL database without using the `codeql database bundle` command:

```yaml
    - name: Zip CodeQL database
      id: codeql-database-bundle
      env:
        LANGUAGE: ${{ matrix.language }}
      run: |
        set -xu
        CODEQL_DATABASE="/home/runner/work/_temp/codeql_databases/$LANGUAGE"
        DATABASE_DIR="/home/runner/work/_temp/codeql_databases"

        for SUB_DIR in log results working; do
          rm -rf $DATABASE_DIR/$SUB_DIR
        done

        CODEQL_DATABASE_ZIP="codeql-database-$LANGUAGE.zip"
        zip -r "$CODEQL_DATABASE_ZIP" "$CODEQL_DATABASE"

        echo "::set-output name=zip::$CODEQL_DATABASE_ZIP"

    - name: Upload CodeQL database
      uses: actions/upload-artifact@v2
      with:
        name: ${{ matrix.language }}-db
        path: ${{ steps.codeql-database-bundle.outputs.zip }}

```
</details>

#### _Full CodeQL Analysis Workflow_
<details>
<summary>Solution</summary>

```yaml
name: "CodeQL"

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '18 10 * * 5'

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write

    strategy:
      fail-fast: false
      matrix:
        language: [ 'go', 'java', 'javascript', 'python' ]

    steps:
    - name: Checkout repository
      uses: actions/checkout@v2

    - name: Initialize CodeQL
      uses: github/codeql-action/init@v1
      with:
        languages: ${{ matrix.language }}
        config-file: ./.github/codeql/codeql-config.yml

    - if: matrix.language == 'java'
      name: Setup Java
      uses: actions/setup-java@v2
      with:
        distribution: 'adopt'
        java-version: '15'

    - if: matrix.language == 'java'
      name: Autobuild
      uses: github/codeql-action/autobuild@v1

    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v1
```
</details>

💡**Looks like we've made it to the end!** 💡
- [Click here to learn more about using the CodeQL CLI](codeql-cli.md)
- [Click here for additional references](api-references.md)
