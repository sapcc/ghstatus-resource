# Git Resource

Tracks the commits in a [github](http://github.com/) repository.
It is a fork from the offical git-resource.
It only emits commits that are in "success" state for the given context.

## Source Configuration

* `owner`: *Required.* The owner of the repository (e.g. `sapcc`).
* `repo`: *Required.* The repositry of the repository (e.g. `kubernikus`).
* `access_token`: *Required.* A github access token that can read the commit status of the given repository
* `base_repo_url`: The base url for the github repository. Default: `https://github.com`
* `base_api_url`: The base url for the github API. Default: `https://api.github.com`
* `context`: The status context to track: Default: `continuous-integration/travis-ci/push`

* `branch`: The branch to track. This is *optional* if the resource is
   only used in `get` steps (default value in this case is `master`).
   However, it is *required* when used in a `put` step.

* `private_key`: *Optional.* Private key to use when pulling/pushing.
    Example:
    ```
    private_key: |
      -----BEGIN RSA PRIVATE KEY-----
      MIIEowIBAAKCAQEAtCS10/f7W7lkQaSgD/mVeaSOvSF9ql4hf/zfMwfVGgHWjj+W
      <Lots more text>
      DWiJL+OFeg9kawcUL6hQ8JeXPhlImG6RTUffma9+iGQyyBMCGd1l
      -----END RSA PRIVATE KEY-----
    ```
    Note: You can also use pipeline templating to hide this private key in source control. (For more information: https://concourse.ci/fly-set-pipeline.html)

* `username`: *Optional.* Username for HTTP(S) auth when pulling/pushing.
  This is needed when only HTTP/HTTPS protocol for git is available (which does not support private key auth)
  and auth is required.

* `password`: *Optional.* Password for HTTP(S) auth when pulling/pushing.

   Note: You can also use pipeline templating to hide this password in source control. (For more information: https://concourse.ci/fly-set-pipeline.html)

* `paths`: *Optional.* If specified (as a list of glob patterns), only changes
  to the specified files will yield new versions from `check`.

* `ignore_paths`: *Optional.* The inverse of `paths`; changes to the specified
  files are ignored.

  Note that if you want to push commits that change these files via a `put`,
  the commit will still be "detected", as [`check` and `put` both introduce
  versions](https://concourse.ci/pipeline-mechanics.html#collecting-versions).
  To avoid this you should define a second resource that you use for commits
  that change files that you don't want to feed back into your pipeline - think
  of one as read-only (with `ignore_paths`) and one as write-only (which
  shouldn't need it).

* `skip_ssl_verification`: *Optional.* Skips git ssl verification by exporting
  `GIT_SSL_NO_VERIFY=true`.

* `git_config`: *Optional.* If specified as (list of pairs `name` and `value`)
  it will configure git global options, setting each name with each value.

  This can be useful to set options like `credential.helper` or similar.

  See the [`git-config(1)` manual page](https://www.kernel.org/pub/software/scm/git/docs/git-config.html)
  for more information and documentation of existing git options.

* `disable_ci_skip`: *Optional.* Allows for commits that have been labeled with `[ci skip]` or `[skip ci]`
   previously to be discovered by the resource.

* `commit_verification_keys`: *Optional.* Array of GPG public keys that the
  resource will check against to verify the commit (details below).

* `commit_verification_key_ids`: *Optional.* Array of GPG public key ids that
  the resource will check against to verify the commit (details below). The
  corresponding keys will be fetched from the key server specified in
  `gpg_keyserver`. The ids can be short id, long id or fingerprint.

* `gpg_keyserver`: *Optional.* GPG keyserver to download the public keys from.
  Defaults to `hkp:///keys.gnupg.net/`.

### Example

Resource configuration for a private repo:

``` yaml
resources:
- name: source-code
  type: git
  source:
    uri: git@github.com:concourse/git-resource.git
    branch: master
    private_key: |
      -----BEGIN RSA PRIVATE KEY-----
      MIIEowIBAAKCAQEAtCS10/f7W7lkQaSgD/mVeaSOvSF9ql4hf/zfMwfVGgHWjj+W
      <Lots more text>
      DWiJL+OFeg9kawcUL6hQ8JeXPhlImG6RTUffma9+iGQyyBMCGd1l
      -----END RSA PRIVATE KEY-----
    git_config:
    - name: core.bigFileThreshold
      value: 10m
    disable_ci_skip: true
```

Fetching a repo with only 100 commits of history:

``` yaml
- get: source-code
  params: {depth: 100}
```

Pushing local commits to the repo:

``` yaml
- get: some-other-repo
- put: source-code
  params: {repository: some-other-repo}
```

## Behavior

### `check`: Check for new commits.

The repository is cloned (or pulled if already present), and any commits
from the given version on are returned. If no version is given, the ref
for `HEAD` is returned.

Any commits that contain the string `[ci skip]` will be ignored. This
allows you to commit to your repository without triggering a new version.

### `in`: Clone the repository, at the given ref.

Clones the repository to the destination, and locks it down to a given ref.
It will return the same given ref as version.

Submodules are initialized and updated recursively.

#### Parameters

* `depth`: *Optional.* If a positive integer is given, *shallow* clone the
  repository using the `--depth` option. Using this flag voids your warranty.
  Some things will stop working unless we have the entire history.

* `submodules`: *Optional.* If `none`, submodules will not be
  fetched. If specified as a list of paths, only the given paths will be
  fetched. If not specified, or if `all` is explicitly specified, all
  submodules are fetched.

* `disable_git_lfs`: *Optional.* If `true`, will not fetch Git LFS files.

#### GPG signature verification

If `commit_verification_keys` or `commit_verification_key_ids` is specified in
the source configuration, it will additionally verify that the resulting commit
has been GPG signed by one of the specified keys. It will error if this is not
the case.

#### Additional files populated

 * `.git/committer`: For committer notification on failed builds.
   This special file `.git/committer` which is populated with the email address
   of the author of the last commit. This can be used together with  an email
   resource like [mdomke/concourse-email-resource](https://github.com/mdomke/concourse-email-resource)
   to notify the committer in an on_failure step.

 * `.git/ref`: Version reference detected and checked out. It will usually contain
   the commit SHA-1 ref, but also the detected tag name when using `tag_filter`.

### `out`: Push to a repository.

Push the checked-out reference to the source's URI and branch. All tags are
also pushed to the source. If a fast-forward for the branch is not possible
and the `rebase` parameter is not provided, the push will fail.

#### Parameters

* `repository`: *Required.* The path of the repository to push to the source.

* `rebase`: *Optional.* If pushing fails with non-fast-forward, continuously
  attempt rebasing and pushing.

* `merge`: *Optional.* If pushing fails with non-fast-forward, continuously
  attempt to merge remote to local before pushing. Only one of `merge` or
  `rebase` can be provided, but not both.

* `tag`: *Optional.* If this is set then HEAD will be tagged. The value should be
  a path to a file containing the name of the tag.

* `only_tag`: *Optional.* When set to 'true' push only the tags of a repo.

* `tag_prefix`: *Optional.* If specified, the tag read from the file will be
prepended with this string. This is useful for adding `v` in front of
version numbers.

* `force`: *Optional.* When set to 'true' this will force the branch to be
pushed regardless of the upstream state.

* `annotate`: *Optional.* If specified the tag will be an
  [annotated](https://git-scm.com/book/en/v2/Git-Basics-Tagging#Annotated-Tags)
  tag rather than a
  [lightweight](https://git-scm.com/book/en/v2/Git-Basics-Tagging#Lightweight-Tags)
  tag. The value should be a path to a file containing the annotation message.

* `notes`: *Optional.* If this is set then notes will be added to HEAD to the
  `refs/notes/commits` ref. The value should be a path to a file containing the notes.

## Development

### Prerequisites

* golang is *required* - version 1.9.x is tested; earlier versions may also
  work.
* docker is *required* - version 17.06.x is tested; earlier versions may also
  work.

### Running the tests

The tests have been embedded with the `Dockerfile`; ensuring that the testing
environment is consistent across any `docker` enabled platform. When the docker
image builds, the test are run inside the docker container, on failure they
will stop the build.

Run the tests with the following command:

```sh
docker build -t git-resource .
```

### Contributing

Please make all pull requests to the `master` branch and ensure tests pass
locally.
