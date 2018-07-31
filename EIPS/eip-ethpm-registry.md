[ Needs table header ... ]

## Abstract
This EIP specifies an interface for publishing to and retrieving assets from smart contract package registries. It is a companion EIP to [1123](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1123.md) which defines a standard for smart contract package manifests.

## Motivation
The goal is to establish a framework that allows smart contract publishers to design and deploy code registries with arbitrary business logic while exposing a set of common endpoints that tooling can use to retrieve assets for contract package consumers.

A clear standard would help the existing EthPM Package Registry evolve from a centralized, single-project community resource into a decentralized multi-registry system whose constituents are bound together by the proposed interface. In turn, these registries could be ENS name-spaced, enabling installation conventions familiar to users of `npm` and other package managers.

**Examples**
```shell
$ ethpm install packages.zeppelin.eth/Ownership
```

```javascript
const SimpleToken = await web3.packaging
                              .registry('packages.ethpm.eth')
                              .getPackage('SimpleToken')
                              .getVersion('^1.1.5');
```

## Specification
The specification describes a small read/write API whose components are mandatory. It allows registries to manage versioned releases using the conventions of [semver](https://semver.org/) without imposing this as a requirement. It assumes registries will share the following structure and conventions:

+ a **registry** is a deployed contract which manages a collection of **packages**.
+ a **package** is a collection of **releases**
+ a **package** is identified by a unique string name within a given **registry**
+ a **release** is identified by a `bytes32` **releaseId** which must be unique for a given package name and release version string pair.
+ a **releaseId** maps to a set of data that includes a **manifestURI** string which describes the location of an [EIP 1123 package manifest](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1123.md). This manifest contains data about the release including the location of its component code assets.
+ a **manifestURI** string contains a cryptographic hash which can be used to verify the integrity of the content found at the URI. The URI format is defined in [RFC3986](https://tools.ietf.org/html/rfc3986).

An example of a package name and release version string pair is:
```shell
"SimpleToken" # package name
"1.0.1"       # version string
```

Implementations are free to choose any scheme for generating a **releaseId**. A common approach would be to hash the strings together as below.

```solidity
// Hashes package name and a release version string
function generateReleaseId(string packageName, string version)
  public
  pure
  returns (bytes32)
  {
    return keccak256(abi.encodePacked(packageName, version));
  }
```

Implementations **must** expose this id generation logic as part of their public `read` API so
tooling can easily map a string based release query to the registry's unique identifier for that release.

**Write API Specification**
The write API consists of a single method, `release`. It passes the registry the package name, a
version identifier for the release, and a URI specifying the location of a manifest which
details the contents of the release.
```solidity
function release(string packageName, string version, string manifestURI) public;
```
**Read API Specification**

The read API consists of a set of methods that allows tooling to extract all consumable data from a registry.

```solidity
// Retrieves all the packages published to a registry.
function getAllPackageNames() public view returns (string[]);

// Retrieves the registry's unique identifier for an existing release of a package.
function getReleaseId(string packageName, string version) public view returns (bytes32);

// Retrieves all release ids for a package
function getAllReleaseIds(string packageName) public view returns (bytes32[]);

// Retrieves package name, release version and URI location data for a release id.
function getReleaseData(bytes32 releaseId) public view
  returns (
    string packageName,
    string version,
    string manifestURI
  );

// Retrieves the release id a registry *would* generate for a package name and version pair
// when executing a release.
function generateReleaseId(string packageName, string version) pure returns (bytes32);

// Declares whether a registry maintains its releases in semver compliant order.
function usesSemver() public pure returns (bool);
```

## Rationale
The proposal hopes to accomplish the following:

+ Define the smallest set of inputs necessary to allow registries to map package names to a set of
release versions while allowing them to use any versioning schema they choose.
+ Provide the minimum set of getter methods needed to retrieve package data from a registry so that registry aggregators can read all of their data.
+ Define a standard query that synthesizes a release identifier from a package name and version pair so that tooling can resolve specific package version requests without needing to query a registry about all of a package's releases.
+ Mandate that registries indicate whether or not they enforce semver so that tooling can determine whether consumer requests for packages at a semver range are possible.

Registries may offer more complex `read` APIs that manage requests for packages within a semver range or at `latest` etc. This EIP is agnostic about how tooling or registries might implement these. It recommends that registries implement [EIP 165](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-165.md) and avail themselves of resources to publish more complex interfaces such as [EIP 926](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-926.md).

## Backwards Compatibility
The standard modifies the interface of the existing EthPM package registry in such a way that the currently deployed version would not comply with standard since it implements only one of the method signatures described in the specification.

## Implementation
A reference implementation of this proposal can be found at the EthPM organization on Github [here](https://github.com/ethpm/escape-truffle).

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
