# checkout-test

This repo is minimal test to show that `actions/checkout@v4.1.4` introduces a new requirement on consumers to use either a newer version of `git` or a newer version of other dependencies that need to handle the `repositoryformatversion` value of `1` in the `.git` directory.

The difference between `actions/checkout@v4.1.4` and `actions/checkout@v4.1.1` is that `sparse-checkout` is disabled explicitly, which depending on the version of `git` being used, results in a `repositoryformatversion` value of `1` being written to the `.git` directory for older versions and a value of `0` for newer versions. What exact `git` version boundary this change occurs at is not clear. The following table shows the combinations of software and their results:

| `actions/checkout` | `git` | `repositoryformatversion` | Test |
|--------------------|-------|---------------------------| ---- |
| `v4.1.1`           | `2.34.1` | `0` | :heavy_check_mark: <https://github.com/hicksjacobp/checkout-test/actions/runs/8848651950/job/24298891141> |
| `v4.1.4`           | `2.34.1` | `1` | :x: <https://github.com/hicksjacobp/checkout-test/actions/runs/8848651950/job/24298890845> |
| `v4.1.4`           | `2.43.2` | `0` | :heavy_check_mark: <https://github.com/hicksjacobp/checkout-test/actions/runs/8848651950/job/24298890623> |

The scenario of `actions/checkout@v4.1.4` and `git@2.34.1` is what has caused us unforeseen pain.
This scenario is exacerbated by the fact that `actions/runner@v3.16.0` comes with `git@2.34.1` and the default CodeQL workflow uses `actions/checkout@v4` (and therefore resolves to `actions/checkout@v4.1.4` as of the time of writing).
Performing an `apt-get install git` does NOT update `git` to a newer version because the base and ancestor images which `actions/runner` is based off of does not use the `apt` feed which has newer versions of git, and is rather left behind at `2.34.1`.

## Resolutions

### Update `git`

In order to update `git` to newer versions, in Ubuntu at least, you have to perform at least these steps:

```bash
apt update
apt install software-properties-common
add-apt-repository -y ppa:git-core/ppa
apt install git
```

See <https://git-scm.com/download/linux> for reference.

> [!TIP]
> Do this in `actions/runner`?

### Update dependencies to handle `repositoryformatversion` of `1`

Personally, I found this issue through referencing `Microsoft.SourceLink.GitHub@1.1.1`. With .NET SDK 8, there is now a `Microsoft.SourceLink.GitHub@8.0.0` package (along with the transitive dependency which can now handle the `repositoryformatversion` of `1`). See <https://github.com/dotnet/sourcelink/pull/772>.

### Downgrade `actions/checkout` to `v4.1.1`

> [!WARNING]
> This may not be possible for your situation, for example using the default CodeQL setup and using a CI infrastructure where updating `git` has not yet been done and is out of your control.

## References

- `actions/runner@v3.16.0` is based off of `mcr.microsoft.com/dotnet/runtime-deps:6.0-jammy` and none of these images do any additional setup for `git`:
  <https://github.com/actions/runner/blob/14cea13ab5e7a5f385d805bf8a9034947d25f1b6/images/Dockerfile>
  <https://github.com/dotnet/dotnet-docker/blob/main/src/runtime-deps/6.0/jammy/amd64/Dockerfile>
- Install instructions for `git`:
  <https://git-scm.com/download/linux>
