# Go upgrade checklist (and how we did it at Lyft)

Upgrading the golang minor version of your service is supposed to be a painless process thanks to the [Go 1 promise of compatibility](https://go.dev/doc/go1compat). However, there can always be small issues around security updates, packages, tools, linters, etc. If you have an uncritical standalone service, it is probably best to upgrade and see if everything works as expected. However, if you are responsible for upgrading a whole bunch of critical services, it is best to have a plan that minimizes any issues or downtime you might face.

I will start with a concise list of steps that are preferable to take and then I will talk about them in more detail based on our experiences at Lyft. In my last role, I was responsible for the Golang ecosystem at Lyft and conducted the upgrade to 1.19 and 1.20. Little disclaimer; I am no longer affiliated with Lyft, therefore I don't have access to the engineering docs that I wrote on this myself anymore. I am writing from memory based on the state of the world in 2023. I don't think I am disclosing any secrets but if anyone from Lyft is reading this and disagrees, please let me know.

## The goals

* Zero or minimal work required by the product engineers (service owners)
* No downtime/issues in prod and minimal issues in staging
* No regression in linter support after the upgrade

## The list

* Read the [release notes](https://go.dev/doc/devel/release)
  * Check Tools/Vet. Changes in the vet tool might give issues even though the code is correct.
  * Do a search with the keyword `existing` to see if there are any caveats on existing code both due to a change in the language itself or the core library.
* It is generally advised to use [golangci-lint](https://golangci-lint.run) for linting. If you are using it upgrade it to the latest version possible.
* Build your binaries with the new version. Go through the build errors if any.
* Run all the unit tests with the new version. Go through the test failures.
* Run golangci-lint on the code for the new version. You can specify the version on the config file `.golangci.yml` or update the go version in `go.mod`. Go through the lint failures and fix/add ignore any issues.
* Upgrade the go version.
* Look for any failures on builds, unit tests, linters in prod. Hopefully there won't be any.
* Update go.mod.

## The details

### Automation

We had around 150 Go services at Lyft by the time I left and the number was increasing. Therefore, builds, tests, lints that are mentioned in the list were all run through scripts. How this is done will depend on your company's infrastructure around image builds, CI/CD, deployment, etc.

Our go service builds were based on a common image which was based on a generic ubuntu image. Once we update this go image to a new version, all the services pick up the update in the next weekly image update. By tapping into this flow, we were able to build all the go services with the new go version in our CI/CD pipeline and look at the results. We were able to do the same thing for unit tests and linting as well.

If you have a similar number of services, try to leverage your existing image build and testing pipeline so this can be done as painlessly as possible.

### Timeline

Once we streamlined the process to what is described here today, 1.20 upgrade was completed in 3-4 weeks with almost no involvement from the service owners and no production issues. Ideally, you can do most of this process at the release candidate stage. The final release candidate is typically released about 1-2 weeks before the minor version release and there is usually not much difference between them, so you can reduce your upgrade timeline to less than 2 weeks if necessary. Golangci-lint is also usually updated in the same timeframe providing full linter support for the new version.

### Example issues

With the introduction of generics at 1.18, many linters lacked support for generics for months. We delayed the upgrade due to this issue. There was talk of trying solve this issue in the upstream ourselves. Fortunately, by the time we seriously started exploring this option, linter support was added and go 1.19 was also released. We eventually upgraded directly to 1.19 from 1.17 but we were around 10 months late.

Other small issues can arise. When sort was rewritten in 1.19, it wasn't giving the same order for equal elements as before. This led to test failures where stable sort wasn't utilized as it should be. The upgrade wasn't the responsible party here, the language kept backwards compatibility and regular sort was never intended to be stable. The developers didn't think through their tests and weren't informed enough about the details of the algorithm.

The linter staticcheck checks for deprecations (SA1019 - Using a deprecated function, variable, constant or field). The deprecations are flagged when the deprecations have happened more than one version ago (when upgrading to 1.N+1, the deprecations from 1.N-1 start getting flagged). This requires going through the lint errors and either updating the code or adding ignore notices to the golangci-lint configuration.

When math/rand package was updated in 1.20, it started automatically seeding the global random number generator with a random value, eliminating the need for the common practice of using the Unix nanosecond as the seed. Even though the old code is still correct, it is best to look out for this kind of changes and clean up the code. Obviously, this would be done after the upgrade.

It should be expected that unforeseen effects of the language changes can delay the upgrade by weeks/months or indefinitely depending on the desire for keeping true to the goals outlines above.

### Patch versions

We usually didn't do any due diligence for patch version upgrades. I haven't seen any issues. Go nuts.

In case you are interested, Lyft had recently released their [Python upgrade playbook](https://eng.lyft.com/python-upgrade-playbook-1479145d52f4) which is a lot more complicated.