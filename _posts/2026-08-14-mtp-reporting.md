---
layout: post
title: Microsoft Testing Platform GitHub and Azure DevOps Reporting
author: Tomasz Cielecki
comments: true
date: 2026-08-14 13:00:00 +0200
tags:
- dotnet
- testing
- devops
- github
- azdo
---

I've [previously written about how cool Microsoft Testing Platform][post] is, which is a set of libraries that a lot of Unit Test frameworks such as xUnit, TUnit, MSTest and more build on. To standardize how the tests are hosted and run and which arguments you can provide to your test runs to generate reports in different formats for both tests and code coverage.

Something new that the team behind the tooling have added is a few libraries to help reporting the test results when they run in GitHub or Azure DevOps. For instance in Azure DevOps Pipelines, when you want to report test results you will have to report this yourself by adding something like this to your pipeline:

```yml
- task: PublishTestResults@2
  inputs:
    testResultsFormat: 'VSTest'
    testResultsFiles: '$(Build.ArtifactStagingDirectory)/Tests/*.trx'
```

In GitHub there is no equivalent to the Test and Coverage tabs on build summaries. But what you could do is generate CTRF reports and use an action like [`ctrf-io/github-test-reporter`][ctrf] to post in PR comments and summaries. Which could look like:

{% raw %}
```yml
- name: Publish Test Report
  uses: ctrf-io/github-test-reporter@e500b992f936420eb633c91644cf10d4d71df700 # v1.1.0
  with:
    report-path: ${{ github.workspace }}/ctrf/*.ctrf.json
    summary-report: true
    github-report: true
    pull-request: true
    update-comment: true
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```
{% endraw %}

Now a lot of this is built straight into the Microsoft Testing Platform tooling, where depending on which environment you want to publish test results to you can add either:

- [Microsoft.Testing.Extensions.AzureDevOpsReport][azdoreport]
- [Microsoft.Testing.Extensions.GitHubActionsReport][ghreport] (currently pre-release)

This will allow you to use the argument `--report-gh` when invoking your tests on GitHub Actions Workflows and the arguments `--report-azdo` and `--publish-azdo-test-results` and `--report-azdo-upload-artifacts files` on runs in Azure DevOps Pipelines.

| Argument | Behavior |
|----------|-----------|
| `--report-gh` | Publishes a summary on GitHub Actions Workflow runs |
| `--report-azdo` | Adds log entries and adds comments on Pull Request code |
| `--publish-azdo-test-results` | Streams test results into the Test tab on a Pipeline run |
| `--report-azdo-upload-artifacts files` | Uploads test results to Pipeline artifacts. This will also include code coverage resuts, but not populate the Test or Coverage tabs |

So running your tests could look as follows for Azure DevOps. In my examples here I am using [TUnit][tunit], but the options could look slightly different if you are using another testing framework. However, the flags from the two new packages remain the same:

{% raw %}
```bash
dotnet test --report-trx --report-trx-filename {asm}_{tfm}_{time}.trx \
            --coverage --coverage-output {asm}_{tfm}_{time}.coverage \
            --coverage-output-format cobertura \
            --report-azdo \
            --publish-azdo-test-results \
            --report-azdo-upload-artifacts files
            --diagnistic
```
{% endraw %}

> Note: there is at the time of writing this post a bit of an issue with the Azure DevOps reporting so you need to add the flag `--diagnostic` for it to not throw a `ObjectDisposedException` and fail your tests. [This will be fixed in next version of the reporter (2.3.4)][bug]

In Azure DevOps make sure to use a testing format that is recognized by Azure DevOps. You can use VSTest (TRX) and JUnit. Also make sure to pass a Access Token to the environment so it can upload the results like:

```yml
script: |
  dotnet test ...
env:
  SYSTEM_ACCESSTOKEN: $(System.AccessToken)
```

For GitHub Actions your test run could look like:

{% raw %}
```bash
dotnet test --report-trx --report-trx-filename {asm}_{tfm}_{time}.trx \
            --coverage --coverage-output {asm}_{tfm}_{time}.coverage \
            --coverage-output-format cobertura \
            --report-gh
```
{% endraw %}

If you want to combine it with CTRF reporting I mentioned before, you have total flexibility and you can still do that.

Your test results in Azure DevOps should appear in the summary like this

[![Screenshot of Azure Pipelines Summary showing Tests and Coverage tabs][azdosummary]][azdosummary]

And in GitHub Actions Workflows it should appear in the summary of the build like this

[![Screenshot of unit test summary in a GitHub actions run][ghsummary]][ghsummary]

For code coverage reports, you will still need to report this yourself using the `PublishCodeCoverageResults@2` task in Azure DevOps, but perhaps in the future they will add support for reporting this through a simple flag too.

You can read the [full announcement on this new Microsoft Testing Platform Reporting on the Microsoft Dev Blogs.][devblogs] and [read more about the mentioned arguments and even more arguments for reporting flaky tests in the Microsoft Learn resources about Testing Reports][learn-reports]

[post]: {% post_url 2025-07-20-ms-test-platform-is-cool %}
[ctrf]: https://github.com/ctrf-io/github-test-reporter "CTRF Test Reporter action"
[azdoreport]: https://www.nuget.org/packages/Microsoft.Testing.Extensions.AzureDevOpsReport "AzureDevOpsReport NuGet"
[ghreport]: https://www.nuget.org/packages/Microsoft.Testing.Extensions.GitHubActionsReport "GitHubActionsReport NuGet"
[tunit]: https://tunit.dev "TUnit testing framework"
[azdosummary]: {{ site.url }}/assets/images/mstestplat2/azdosummary.png "Screenshot of Azure Pipelines Summary showing Tests and Coverage tabs"
[ghsummary]: {{ site.url }}/assets/images/mstestplat2/gh-actions.png "Screenshot of unit test summary in a GitHub actions run"
[devblogs]: https://devblogs.microsoft.com/dotnet/microsoft-testing-platform-reporting/ "Microsoft Dev Blog about Microsoft Testing Platform Reporting"
[learn-reports]: https://learn.microsoft.com/en-us/dotnet/core/testing/microsoft-testing-platform-test-reports#azure-devops-reports "Microsoft Learn page about Test Reports"
[bug]: https://github.com/microsoft/testfx/issues/10191 "ObjectDisposedException bug report"