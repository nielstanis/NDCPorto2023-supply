# Securing your .NET application software supply-chain, the practical approach! 

With our complete software development process becoming more complex we also got a lot more security problems to deal with. What starts with code and ends with releasing/deploying software is also being referred at as the software supply chain. Over the last years we've seen some big security incidents tied to the software supply chain and the software industry acknowledged there was a need for action. And today there is a lot to choose from, but what will be the most effective things to do?

What to expect; we're doing some introductions/overviews on the different area's followed by labs. We're going to focus on both producing and consuming software. The commands used in this workshop will be running on either MacOSX and/or Linux (WSL) with Windows alternatives; but we'll work it out once we get there. 

# Introducing DocGenerator Project

A library called `docgenerator` that has been published on feedz.io can be used to process documents. This is a library created for the purpose of this workshop and we're going te looking into securing the supply chain of it from _producing_ and _consuming_ perspective.

## Lab 1 - Signing artifacts with SigStore

In order to do this locally you need to get cosign installed:
https://edu.chainguard.dev/open-source/sigstore/cosign/how-to-install-cosign/ 

Or grab the platform specific binaries from the last releases for:
- https://github.com/sigstore/cosign/releases
- https://github.com/sigstore/rekor/releases (only need CLI)
- https://github.com/jqlang/jq/releases

Then we're going to do the following things:

- `echo Bom dia NDC Porto 2023! >> ndcporto.txt`
- `cosign sign-blob --rekor-url https://rekor.sigstore.dev --oidc-issuer https://oauth2.sigstore.dev/auth ndcporto.txt` We need the ID returned for the next step. 
- `rekor-cli get --log-index IDHERE --format json | jq > output.json`
- MacOSX/Linux
  - `cat output.json | jq -r .Body.HashedRekordObj.signature.content > ndcporto.txt.sig`
  - `cat output.json | jq -r .Body.HashedRekordObj.signature.publicKey.content | base64 -d > pub.crt`
  - `openssl x509 -noout -text -in pub.crt`
- Windows
  - `type output.json | jq -r .Body.HashedRekordObj.signature.content > ndcporto.txt.sig`
  - `type output.json | jq -r .Body.HashedRekordObj.signature.publicKey.content > pub.b64`
  - `certutil -decode pub.b64 pub.crt`
  - View certificate details in Windows Explorer
- Did you notice how long the certificate was valid?
- Now verify the blob with following command, replace certificate-identity with yours and oidc with the one you've chosen:
  - GitHub: `cosign verify-blob --cert pub.crt --certificate-identity nielstanis@live.nl --certificate-oidc-issuer https://github.com/login/oauth --signature ndcporto.txt.sig ndcporto.txt`
  - Google: `cosign verify-blob --cert pub.crt --certificate-identity niels.tanis@gmail.com --certificate-oidc-issuer  https://accounts.google.com --signature ndcporto.txt.sig ndcporto.txt`
  - GitHub: `cosign verify-blob --cert pub.crt --certificate-identity nielstanis@live.nl --certificate-oidc-issuer https://login.microsoftonline.com --signature ndcporto.txt.sig ndcporto.txt`
- Now change the `ndcporto.txt` file and verify the blob again.

# Git Commit Signing

It's a real good idea to start git commit signing, resulting in an additional factor that can be verified. If in case of credential theft it can be as an extra indicator if something bad has happened. 

We're not going to add it to our current workflow because it contains a lot of loop holes and constrains and it will probably fail. 

- https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits 
- https://www.hanselman.com/blog/how-to-setup-signed-git-commits-with-a-yubikey-neo-and-gpg-and-keybase-on-windows 
- https://blog.1password.com/git-commit-signing/

## Lab 2 - Setup repository, build and git signing with GitSign

In order to start create a fork of the `https://github.com/nielstanis/docgenerator` repository.
Make sure you're enabling Actions and execute the current .NET build available to generate the output. 
Clone the repo to local location on your machine. 

Now we need to have GitSign installed and runnable. If you have not done that go to the url below and download the platform based binaries or install with one of the package managers for your system:
https://github.com/sigstore/gitsign/releases

Then in our git directory for the `docgenerator` we're going to configure it for the _single_ repository.

```sh
cd /path/to/my/repository
git config --local commit.gpgsign true  # Sign all commits
git config --local tag.gpgsign true  # Sign all tags
git config --local gpg.x509.program gitsign  # Use gitsign for signing
git config --local gpg.format x509  # gitsign expects x509 args
```

And now we're going push some changes to the repo.

```sh
echo Bon dia NDC Porto 2023! >> ndcporto.txt
git add .
git commit -S -m "Hello NDC Porto!" #starts OICD flow in external browser, walk through
git push origin main
```

Unfortunately the `verified` won't be shown in case of use of `gitsign`, this is because GitHub does not trust the SigStore CA and the keys are also valid for a short time it should validate the x509 certificate instead. GitSign can validate the signature done more information on that can be done with the `gitsign --verify` command.

# Reproducible Builds

## Lab 3 - .NET reproducibility

- Create new console with the following CLI call : `dotnet new console -n ConsoleApp`
- Rename the folder `ConsoleApp` to `ConsoleApp2`
- Execute `dotnet new console -n ConsoleApp` again that will create a new folder
- Now build both of the projects
  - `dotnet build ConsoleApp\ConsoleApp.csproj`
  - `dotnet build ConsoleApp2\ConsoleApp.csproj`
- What will the output binaries look like? 
  - MacOSX/Linux: `diff ConsoleApp/bin/Debug/net7.0/ConsoleApp.dll ConsoleApp2/bin/Debug/net7.0/ConsoleApp.dll`
  - Windows: `fc ConsoleApp/bin/Debug/net7.0/ConsoleApp.dll ConsoleApp2/bin/Debug/net7.0/ConsoleApp.dll`
- Maybe you can decompile (e.g. JetBrains Rider) and look into what it looks like?
- Run string on the binaries and create text files to compare with e.g. VS Code or any other preferred tool
  - MacOSX/Linux:
    - `strings ConsoleApp/bin/Debug/net7.0/ConsoleApp.dll >> ConsoleAppStrings`
    - `strings ConsoleApp2/bin/Debug/net7.0/ConsoleApp.dll >> ConsoleApp2Strings`
  - Windows:
    - `more < ConsoleApp/bin/Debug/net7.0/ConsoleApp.dll | findstr "." >> ConsoleAppStrings`
    - `more < ConsoleApp2/bin/Debug/net7.0/ConsoleApp.dll | findstr "." >> ConsoleApp2Strings`
  - What are the differences? If you have VS Code on your machine consider doing the following:
    `code -r --diff ConsoleAppStrings ConsoleApp2Strings`
- Alter both `.csproj` files and add `<DebugType>None</DebugType>` to it.
- Build both projects and check the strings output again same way described above. 

## Roslyn Compiler Deterministic

Based on given `inputs` the produced `outputs` should be the same. This has been done since Roslyn v1.1

https://github.com/dotnet/roslyn/blob/master/docs/compilers/Deterministic%20Inputs.md

## Lab 4 - Dotnet.ReproducibleBuild

Now let us add two packages to our project that will help out with setting certain project settings that can be found here:

https://github.com/dotnet/reproducible-builds/blob/main/README.md

Add `DotNet.ReproducibleBuilds` and `DotNet.ReproducibleBuilds.Isolated` (SDK) and commit the changes initialing the build to produce a new artifact. 

## NuGet Proposed Design
   
https://github.com/dotnet/designs/blob/main/accepted/2020/reproducible-builds.md
https://github.com/dotnet/designs/pull/171 

# SBOM - Why Needed

## Lab 5 - CycloneDX

Now we're going to add CycloneDX to our build pipeline. The existing Action shared in the marketplace is not working properly, that's why we're going to add the dotnet cli library and use that. Take the following steps.

- Add `dotnet tool install --global CycloneDX` as next step with name `Install CycloneDX in dotnet cli` to our build after we've done our `dotnet test`.
- Add `dotnet CycloneDX docgenerator.csproj -j -o . ` as next step with name `CycloneDX .NET Generate SBOM`.
- We need to add the `bom.json` file to our stored outputs of pipeline, by creating multiple files in `actions/upload-artifact` step

Your build definition will look similar to this: 

```yaml
name: .NET

on:
  workflow_dispatch:

jobs:
  build:

    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 7.0.x
    - name: Restore dependencies
      run: dotnet restore
    - name: Build
      run: dotnet build -c Release -o .
    - name: Test
      run: dotnet test --no-build --verbosity normal
    - name: Install CycloneDX in dotnet cli
      run: dotnet tool install --global CycloneDX
    - name: CycloneDX .NET Generate SBOM
      run: dotnet CycloneDX docgenerator.csproj -j -o . 
    - name: Upload a Build Artifact
      uses: actions/upload-artifact@v3.1.0
      with:
          path: | 
              docgenerator.1.0.0.nupkg
              bom.json
          name: DocGenerator
          if-no-files-found: error
          retention-days: 5
```

Now commit+sign+push the changes to GitHub, get pipeline to run and download `bom.json` and look into it's contents. 

Are there any issues in the output? Consider checking `dotnet list package --vulnerable`. Are you sure you did see _all_ that's inside?

# SLSA Generator Provenance Generator

## Lab 6 - SLSA Provenance

We're going to use the default Provenance Generator for GitHub Actions. We need to make sure to do the following things to our build pipeline so it will look like the example output you find below. 

- Make sure permissions get added to the `runs-on` section that will allow `write` to `packages` and `id-token`. 
- Also we need to define `outputs` that will have the name of `hashes` and gets `${{ steps.hash.outputs.hashes }}`.
- Next up is going to generate sha256 hashes of the artifacts we want to tie to our generated provenance:

```yaml
- name: Generate hashes
    shell: bash
    id: hash
    run: |
    # sha256sum generates sha256 hash for all artifacts.
    # base64 -w0 encodes to base64 and outputs on a single line.
    # echo "::set-output name=hashes::$(sha256sum docgenerator.1.0.0.nupkg bom.json| base64 -w0)"
    echo "hashes=$(sha256sum docgenerator.1.0.0.nupkg bom.json | base64 -w0)" >> $GITHUB_OUTPUT
```

- The `hashes` are combined and set as output in base64 encoded string. 
- After that we're adding the provenance generation job which will look as follows:

```yaml
provenance:
needs: [build]
permissions:
    id-token: write # For signing.
    contents: write # For asset uploads.
    actions: read # For the entrypoint.
uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@main
with:
    base64-subjects: "${{ needs.build.outputs.hashes }}"
    compile-generator: true
```
- This will generate the output and tie the previously generated hashes to it. It will under the hood also sign the artifact wih a project called SigStore which we'll touch later. 
- The output build pipwline can be committed+pushed to our GitHub repo to start the pipeline and create artifacts. For reference the build pipeline should look following;

```yaml
name: .NET

on:
  workflow_dispatch:

jobs:
  build:

    runs-on: ubuntu-latest
    permissions:
      packages: write
      id-token: write
    outputs:
      hashes: ${{ steps.hash.outputs.hashes }}

    steps:
    - uses: actions/checkout@v3
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 7.0.x
    - name: Restore dependencies
      run: dotnet restore
    - name: Build
      run: dotnet build -c Release -o .
    - name: Test
      run: dotnet test --no-build --verbosity normal
    - name: Install CycloneDX in dotnet cli
      run: dotnet tool install --global CycloneDX
    - name: CycloneDX .NET Generate SBOM
      run: dotnet CycloneDX docgenerator.csproj -j -o . 
    - name: Generate hashes
      shell: bash
      id: hash
      run: |
        # sha256sum generates sha256 hash for all artifacts.
        # base64 -w0 encodes to base64 and outputs on a single line.
        # echo "::set-output name=hashes::$(sha256sum docgenerator.1.0.0.nupkg bom.json| base64 -w0)"
        echo "hashes=$(sha256sum docgenerator.1.0.0.nupkg bom.json | base64 -w0)" >> $GITHUB_OUTPUT
    - name: Upload a Build Artifact
      uses: actions/upload-artifact@v3.1.0
      with:
          path: | 
              docgenerator.1.0.0.nupkg
              bom.json
          name: DocGenerator
          if-no-files-found: error
          retention-days: 5
  
  provenance:
    needs: [build]
    permissions:
      id-token: write # For signing.
      contents: write # For asset uploads.
      actions: read # For the entrypoint.
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@main
    with:
      base64-subjects: "${{ needs.build.outputs.hashes }}"
      compile-generator: true
```

- Next up is to download the generated artifact, zip containing SBOM and NuGet and SLSA provenance and check out it's contents.
- Extract payload and decode from `multiple.intoto.json` by executing
  - MacOSX/Linux: `cat multiple.intoto.jsonl | jq -r .payload | base64 -d > payload.json`
  - Windows: `type multiple.intoto.jsonl | jq -r .payload > payload.b64` and `certutil -decode payload.b64 payload.json`
- Generate SHA256 has of both SBOM and NuGet Package and cross check:
  - MacOSX/Linux: `shasum -a 256 docgenerator.1.0.0.nupkg bom.json` 
  - Windows: `certUtil -hashfile docgenerator.1.0.0.nupkg SHA256` and `certUtil -hashfile bom.json SHA256`
- It also signs the generated provenance that can be verified against the public information on SigStore system, which is described earlier. 

# Secure Supply Chain Consumption Framework (S2C2F)

The Secure Supply Chain Consumption Framework (S2C2F) is a security assurance and risk reduction process that is focused on securing how developers consume open source software. As a Microsoft-wide initiative since 2019, the S2C2F provides security guidance and tools throughout the developer inner- loop and outer-loop processes that have played a critical role in defending and preventing supply chain attacks through consumption of open source software across Microsoft.

https://www.microsoft.com/en-us/securityengineering/opensource/osssscframeworkguide 
https://github.com/ossf/s2c2f/blob/main/specification/Secure_Supply_Chain_Consumption_Framework_(S2C2F).pdf 

# ASP.NET MVC that uses DocGenerator

## Lab 7 - Create ASP.NET MVC uses DocGenerator and runs on Docker

- Create new MVC site with name `docwebgen` by calling `dotnet new mvc -n docwebgen`.
- Add following `nuget.config` file to project:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
 <packageSources>
    <!--To inherit the global NuGet package sources remove the <clear/> line below -->
    <clear />
    <add key="docgen-packages" value="https://f.feedz.io/fennec/docgenerator/nuget/index.json" /> 
    <add key="nuget" value="https://api.nuget.org/v3/index.json" />
</packageSources>
</configuration>
```

- Add the `docgenerator` package by calling `dotnet add pacakge docgenerator`
- Create a `Dockerfile` that contains the following. 

```Dockerfile
# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build-env
WORKDIR /app

# Copy csproj and restore as distinct layers
COPY *.csproj ./
COPY nuget.config ./
RUN dotnet restore --configfile "./nuget.config"

# Copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o out

# Build runtime image
FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY --from=build-env /app/out .
ENTRYPOINT ["dotnet", "docwebgen.dll"]
```

- Create a new `.gitignore` file by calling `dotnet new gitignore`.
- Initialize new Git repo `git init`.
- Optional; enable `gitsign` for this repo as described earlier. 
- Add changes to it `git add .` and commit `git commit -m "Website" -S` (lose `-S` if your not signing the commit). 
- Create a new GitHub Repo and call it `DocWebGen` and push changes to that one. 
- We need to create a GitHub Actions pipeline (main.yml) that looks following, make sure to put in the right repo name to store the created image by replacing `GITHUBACCOUNT` with your own.
- Right now we work on tags; we should work on sha hashes of images and actions!

```yaml
name: container

on:
  push:
    tags:        
      - v1             # Push events to v1 tag
      - v1.*           # Push events to v1.0, v1.1, and v1.9 tags
  workflow_dispatch:
    
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      packages: write
      id-token: write

    steps:
      - uses: actions/checkout@v1

      - name: Login to GitHub
        uses: docker/login-action@v1.9.0
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: build+push
        uses: docker/build-push-action@v2.7.0
        with:
          context: .
          push: true
          tags: ghcr.io/GITHUBACCOUNT/docwebgen:latest
```

# Docker image + Syft SBOM

In 2022 Docker released SBOM support that is based on a project called `Syft` that will generate SBOM data of a container and it's contents.
We can add this pretty easily to our build pipeline based on the following lab.

## Lab 8 Anchor Syft

```yaml
name: container

on:
  push:
    tags:        
      - v1             # Push events to v1 tag
      - v1.*           # Push events to v1.0, v1.1, and v1.9 tags
  workflow_dispatch:
    
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      packages: write
      id-token: write

    steps:
      - uses: actions/checkout@v3.6.0

      - name: Login to GitHub
        uses: docker/login-action@v2.2.0
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: build+push
        uses: docker/build-push-action@v4.1.1
        with:
          context: .
          push: true
          tags: ghcr.io/GITHUBACCOUNT/docwebgen:latest
          
      - uses: anchore/sbom-action@v0
        with:
          image: ghcr.io/GITHUBACCOUNT/docwebgen:latest
```

# Ubuntu Chiseled and Chainguard Wolfi

Consider moving to distro-less Linux distributions to reduce attack surface this can be achieved with Ubuntu Chiseled and Chainguards Wolfi. 

## Lab 9 Cosign Container

Now we need to add both the signing and verification with Cosign into our build pipeline. This is seen in the steps with `cosign sign` and `cosign verify` below. Make sure you replace `GITHUBACCOUNT` with your own one.  

```yaml
name: container

on:
  push:
    tags:        
      - v1             # Push events to v1 tag
      - v1.*           # Push events to v1.0, v1.1, and v1.9 tags
  workflow_dispatch:
    
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      packages: write
      id-token: write

    steps:
      - uses: actions/checkout@v3.6.0

      - name: Login to GitHub
        uses: docker/login-action@v2.2.0
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: build+push
        uses: docker/build-push-action@v4.1.1
        with:
          context: .
          push: true
          tags: ghcr.io/GITHUBACCOUNT/docwebgen:latest

      - uses: sigstore/cosign-installer@v3.1.1

      - name: Sign the images
        run: |
          cosign sign --yes \
            ghcr.io/GITHUBACCOUNT/docwebgen:latest
  
      - name: Verify the pushed tags
        run: |
          cosign verify ghcr.io/GITHUBACCOUNT/docwebgen:latest \
          --certificate-identity https://github.com/GITHUBACCOUNT/docwebgen/.github/workflows/main.yml@refs/heads/main \
          --certificate-oidc-issuer https://token.actions.githubusercontent.com
          
      - uses: anchore/sbom-action@v0
        with:
          image: ghcr.io/GITHUBACCOUNT/docwebgen:latest
```

We can get cosign (if installed locally) to verify as well, it also makes sense to look into the generated data by doing the following:

- `rekor-cli get --log-index IDREG --format json | jq > output.json`
- `cat output.json | jq -r .Body.HashedRekordObj.signature.publicKey.content | base64 -d > pub.crt`
- `openssl x509 -noout -text -in pub.crt`
- `cosign verify ghcr.io/GITHUBACCOUNT/docwebgen:latest`

# Hacked now what?

## Lab 10

Maybe you should try to figure out how it got `Hacked` :D
