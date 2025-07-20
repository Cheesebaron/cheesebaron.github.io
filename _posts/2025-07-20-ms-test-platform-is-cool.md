---
layout: post
title: Microsoft Testing Platform is cool!
author: Tomasz Cielecki
comments: true
date: 2025-07-20 13:00:00 +0100
tags:
- testing
- dotnet
- devops
---

I love [xUnit][xunit] and I use it for most of my testing. With [xUnit][xunit] v3 the possibility of running the tests was added without the need of using any other external tools like console runners, `dotnet test` or [VSTest][vstest]. So a [xUnit][xunit] test suite is just a executable itself.
Not long after [xUnit][xunit] v3 were released I discovered [Microsoft Testing Platform][mstestplatdoc], which achieves something very similar. According to their own [docs it "\[..\]is a lightweight and portable alternative to VSTest for running tests in all contexts"][mstestplatdoc].

The cool thing about Microsoft Test Platform is that it provides this for all kinds of test libraries such as [xUnit][xunit], [NUnit][nunit], [Expecto][expecto], [MSTest][mstest] and [TUnit][tunit]. So the Microsoft Testing Platform will work for you regardless of unit testing library you use, and regardless of the library supporting running standalone or not, since Microsoft Testing Platform will provide the runner as well.

I still use Microsoft Testing Platform with [xUnit][xunit] since they coorporate very well, and using them together simplifies a few things which required additional setup and libraries. So recently, I've been converting a few test suites to leverage Microsoft Testing Platform so let me share a few cool things I like about it.

So if you already have a Unit Test suite you can opt into the Microsoft Testing Platform bits by adding the following to your csproj file:

```xml
<UseMicrosoftTestingPlatformRunner>true</UseMicrosoftTestingPlatformRunner>
```

This way you can run the tests with `dotnet run`, now `xUnit.v3` already has this built in, but opting into the Testing Platform will then use the Testing Platform runner instead. If you still want to support running with `dotnet test` you can also add:

```xml
<TestingPlatformDotnetTestSupport>true</TestingPlatformDotnetTestSupport>
```

You can also at this point remove references to packages such as, because the Microsoft Testing Platform also handles integration with your IDE:

```xml
<PackageReference Include="xunit.runner.visualstudio">
```

## Report formats

One cool thing about using Microsoft Testing Platform with xUnit is the wide variety of output types you get for reports. For instance some other tools I use such as SonarCloud, only supports [xUnit][xunit], [VSTest][vstest] of [NUnit][nunit] report types.
While some other tools I use require a CTRF type report. This is no issue and can be easily configured when executing the tests with commandline arguments. To export a TRX ([VSTest][vstest]) and CTRF report I do:

```bash
dotnet run --project UnitTest/UnitTest.csproj -- \
  --report-xunit-trx --report-xunit-trx-filename UnitTestReport.xml \
  --report-ctrf --report-ctrf-filename UnitTestReport.ctrf.json
```

This gives me both formats which I then can use for whatever other tools I use. Great!

## Code coverage

Gathering code coverage with Microsoft Testing Platform is also made super easy, previously I was relying on another package called `coverlet.collector`, however this package is designed for [VSTest][vstest] and does not work with Microsoft Testing Platform. Instead you can use their own extension [`Microsoft.Testing.Extensions.CodeCoverage`][codecoverageext], it can export code coverage reports in a VS binary format, XML and cobertura.

So in a very similar fashion like the various types of test report formats, adding code coverage after having added the NuGet package is done in a very similar fashion. So piecing everything together:

```bash
dotnet run --project UnitTest/UnitTest.csproj -- \
  --report-xunit-trx --report-xunit-trx-filename UnitTestReport.xml \
  --report-ctrf --report-ctrf-filename UnitTestReport.ctrf.json \
  --coverage --coverage-output CoverageReport.coverage --coverage-output-format cobertura
```

Super convenient. To generate a report locally you can still use the awesome [.NET tool `dotnet-reportgenerator-globaltool`][reportgenerator] which can generate very nice HTML and markdown reports among many other exports.

## Reporting test summary with GitHub Actions

So as I mentioned previously being able to export to various formats is a very powerful feature. One thing I use the CTRF reports for is to post Unit Test summaries on GitHub Actions runs. This summary looks something like this:

[![Screenshot of unit test summary in a GitHub actions run][ghactions-summary]][ghactions-summary]

This is powered by the [`ctrf-io/github-test-reporter` action][gh-test-reporter] configured like this:

```yaml
- name: Publish Test Report
  uses: ctrf-io/github-test-reporter@073c73100796cafcbfdc4722c7fa11c29730439e #v1.0.18
  with:
    report-path: ${{ github.workspace }}/ctrf/*.ctrf.json
    summary-report: true
    github-report: true
    pull-request: true
    update-comment: true
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

I really like the flexibility and capabilities that MS Testing Platform with xUnit gives me. I was able to migrate to Microsoft Testing Platform pretty easily and simplify some of my CI setup. Also the fact that you would be able to have tests using various frameworks and provide the same arguments to the runner is something that seems pretty interesting too. Since Microsoft Testing Platform would be taking care of the running and argument processing. This would allow someone to easily understand the setup but also swap it out for something else without having to mess with the arguments.

Another outcome from writing this article is getting to know of [TUnit][tunit], which seems to take a slightly different approach to xUnit and other frameworks and base fully on top of Microsoft Testing Platform instead of adopting it. I might have to check this one out in the near future and see what it can do! The promise of heavily relying on Source Generators, supporting NativeAOT and fully trimmable sounds great.

[xunit]: https://xunit.net/
[vstest]: https://github.com/microsoft/vstest
[nunit]: https://nunit.org/
[expecto]: https://github.com/haf/expecto
[mstest]: https://github.com/microsoft/testfx
[tunit]: https://github.com/thomhurst/TUnit
[mstestplatdoc]: https://learn.microsoft.com/en-us/dotnet/core/testing/microsoft-testing-platform-intro
[reportgenerator]: https://github.com/danielpalme/ReportGenerator
[codecoverageext]: https://www.nuget.org/packages/Microsoft.Testing.Extensions.CodeCoverage
[ghactions-summary]: {{ site.url }}/assets/images/mstestplat/gh-actions.png "Screenshot of unit test summary in a GitHub actions run"
[gh-test-reporter]: https://github.com/ctrf-io/github-test-reporter