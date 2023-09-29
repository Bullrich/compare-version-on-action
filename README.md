# Compare version on action

Action to compare the docker image version inside a `action.yml` file. Intended to be used in other GitHub actions’ as a check.

## Installation
You need to have actions/checkout in your job and then add a step to your GitHub action to use this step:
```yaml
- uses: actions/checkout@v3.3.0
- name: Compare versions
  id: comparison
  uses: Bullrich/compare-version-on-action@main
  with:
    version: 0.0.1
```

The action requires an input named `version`. 
It also produces an action named `version` if the version matches.
If the `image` field inside the `runs` section is `Dockerfile`, the image will not fail, but it won’t produce an output (useful when you don’t want to fail the process but a chained process requires such a field).
## Use case

GitHub actions which uses a `docker` image can build their `Dockerfile` on every run, or, they can use a prebuilt image to save time.

These images are usually stored in [Docker Hub](https://hub.docker.com/) or in [GitHub Packages](https://github.com/features/packages).

An action file that uses a prebuilt image, usually have a configuration like this:
```yaml
runs:
  using: 'docker'
  image: 'docker://ghcr.io/bullrich/compare-version-on-action/action:0.0.2'
```

What this action does, is, it extracts the `0.0.2` in `docker://ghcr.io/bullrich/compare-version-on-action/action:0.0.2`, and compares it against a string. If it returns the same string, it generates an output. If not, it fails.

### Example
Let’s say that you have an action that has a `package.json`, and you want to have the same version in the `package.json` than in the deployed Docker image. And if the version is new, you want to make a new release.

This would be an example where we can use this image:
```yaml
  compare-versions:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3.3.0
      # Get the version from the .version field in the package.json
      - name: Extract package.json version
        id: package_version
        run: echo "VERSION=$(jq '.version' -r package.json)" >> $GITHUB_OUTPUT
        # Call this action and gives the version extracted from the package.json
      - name: Compare versions
        id: comparison
        uses: Bullrich/compare-version-on-action@main
        with:
          version: ${{ steps.package_version.outputs.version }}
        # Use an action to verify if such version already has a tag
      - uses: mukunku/tag-exists-action@v1.4.0
        id: checkTag
        with: 
          tag: ${{ steps.comparison.outputs.version }}
        # If the version does not exist, release a new version
      - name: Tag version and create release
        if: steps.checkTag.outputs.exists == 'false'
        run: gh release create $VERSION --generate-notes
        env:
          VERSION: ${{ steps.comparison.outputs.version }}
          GH_TOKEN: ${{ github.token }}
```

You can see that we are extracting the field `.version` from the `package.json` by using `jq`, and then we are using that as an input.

Once we have the result, we extract the version and we check if the tag exists, and if it does not exist, we create a new release.
