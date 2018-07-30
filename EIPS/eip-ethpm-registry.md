[ Needs table header ... ]

## Abstract
This EIP specifies an interface for publishing to and retrieving assets from smart contract package registries. It is a companion EIP to [1123](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1123.md) which defined a standard for smart contract package manifests.

## Motivation
The goal is to establish a framework that allows smart contract publishers to design and deploy code registries of arbitrary complexity which expose standard endpoints to tooling that retrieves assets for contract package consumers.

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
The specification describes a small read/write API whose components are mandatory. It encourages (without enforcing) the management of versioned releases using the conventions of [semver](https://semver.org/). It assumes registries will share the following structure and encoding conventions:

+ a **registry** is a deployed contract which manages a collection of **packages**.
+ a **package** is a collection of **releases**
+ a **package** is identified by a unique string name within a given **registry**
+ a **release** is identified by a bytes32 **releaseHash** which is the keccak256 hash of the following:
  + the keccak256 hash of a package's string name *WITH*
  + the keccak256 hash of its semver components
    + uint32 major
    + uint32 minor
    + uint32 patch
    + string preRelease
    + string build
+ a **releaseHash** maps to a set of data that includes a **manifestURI** string which describes the location of an [EIP 1123 package manifest](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1123.md). This manifest contains data about the release including the location of its component code assets.
+ a **manifestURI** string contains a cryptographic hash which can be used to verify the integrity of the content found at the URI. The URI format is defined in [RFC3986](https://tools.ietf.org/html/rfc3986).

The canonical way to generate a  **releaseHash** is the using the following methods (in Solidity):
```solidity
// Hashes package name
function hashPackageName(string packageName)
  public
  pure
  returns (bytes32)
  {
    return keccak256(abi.encodePacked(packageName));
  }

// Hashes version components
function hashReleaseVersion( uint32 major, uint32 minor, uint32 patch, string preRelease, string build)
  public
  pure
  returns (bytes32)
  {
    return keccak256(abi.encodePacked(major, minor, patch, preRelease, build));
  }

// Hashes package name hash and version components hash together
function hashRelease(bytes32 packageNameHash, bytes32 releaseVersionHash)
  public
  pure
  returns (bytes32)
  {
    return keccak256(abi.encodePacked(packageNameHash, releaseVersionHash));
  }
```
(See *Rationale* below for more information about the purpose of this hashing strategy.)

**Write API**
The write API consists of a single method, `release` which passes the registry the release information described above and allows it to create a unique, retrievable, semver compliant record of a versioned code package.
```solidity
function release(
    string name,
    uint32 major,
    uint32 minor,
    uint32 patch,
    string preRelease,
    string build,
    string manifestURI
  )
    public
    returns (bool);
```
**Read API Specification**

The read API consists of a minimal set of methods that allows tooling to extract all consumable data from a registry.

```solidity
// Retrieves all packages published to a registry
function getAllPackageNames() public view returns (string[]);

// Retrieves all releases for a given package
function getAllPackageReleaseHashes(string name) public view returns (bytes32[]);

// Retrieves version and manifestURI data for a given release hash
function getReleaseData(bytes32 releaseHash) public view
  returns (
    uint32 major,
    uint32 minor,
    uint32 patch,
    string preRelease,
    string build,
    string manifestURI
 );
```

## Rationale
The proposal is meant to accomplish the following:
+ Establish a publication norm that helps registries implement semver and store package data in mappings that reflect a two-tiered hierarchy of packages that are collections of releases. This is the rationale behind the two-phased hashing of package and version components together into a single release hash identifier.
+ Provide the minimum set of getter methods needed to retrieve all package data from a registry so that registry aggregators can read all of their data.
+ Define a standard way of generating a release hash so that tooling can resolve specific package version requests *without* needing to query a registry about its entire contents.

In practice registries may offer more complex `read` APIs that manage consumers requests for packages within a semver range or at `latest` etc. This EIP is agnostic about how tooling or registry contracts implement these. It recommends that registries implement [EIP 165](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-165.md) and avail themselves of resources to publish more complex interfaces such as [EIP 926](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-926.md).

## Backwards Compatibility
The standard simplifies the interface of the existing EthPM package registry in such a way that the currently deployed version would not comply with proposed standard. Specifically, the deployed version lacks the `getAllPackageNames` method.

## Implementation
A reference implementation of the proposed standard can be found at the EthPM organization on Github [here](https://github.com/ethpm/escape-truffle).

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).