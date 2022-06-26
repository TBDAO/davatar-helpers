# ENS Avatar field format specification
This document describes a process for retrieving avatar URIs from ENS, several [URI](https://datatracker.ietf.org/doc/html/rfc3986) schemes for the ENS 'avatar' text field, and how they should be interpreted by clients wishing to display a user's avatar image.

## Retrieving the avatar URI
The process for retrieving the avatar URI depends on whether the client has an Ethereum address or an ENS name to start with.

### ENS Name
To determine the avatar URI for an ENS name, the client MUST first look up the resolver for the name and call `.text(namehash(name), 'avatar')` on it to retrieve the avatar URI for the name.

The client MUST treat the absence of a resolver, an revert when calling the `addr` method on the resolver, or an empty string returned by the resolver identically, as a failure to find a valid avatar URI.

### Ethereum Address
To determine the avatar URI for an Ethereum address, the client MUST reverse-resolve the address by querying the ENS registry for the resolver of `<address>.addr.reverse`, where `<address>` is the lowercase hex-encoded Ethereum address, without leading '0x'. Then, the client calls `.text(namehash('<address>.addr.reverse'), 'avatar')` to retrieve the avatar URI for the address.

If a resolver is returned for the reverse record, but calling `text` causes a revert or returns an  empty string, the client MUST call `.name(namehash('<address>.addr.reverse'))`. If this method returns a valid ENS name, the client MUST:
1. Validate that the reverse record is valid, by resolving the returned name and calling `addr` on the resolver, checking it matches the original Ethereum address.
2. Perform the process described under 'ENS Name' to look for a valid avatar URI on the name.

A failure at any step of this process MUST be treated by the client identically as a failure to find a valid avatar URI.

## General Format
The 'avatar' text field MUST be formatted as a URI. Clients MUST ignore URI types they do not recognise, treating them the same as if no value was set for the field.

## Image Types
Clients MUST support images with mime types of  `image/jpeg`, `image/png`, and `image/svg+xml`. Clients MAY support additional image types.

## URI Types
All clients SHOULD support the URI schemes defined below. They MAY implement additional schemes not defined in this specification.

### `https`
If an https URI is provided, it MUST resolve to an avatar image directly. https URLs MUST NOT resolve to HTML pages, metadata, or other content containing the avatar image.

### `ipfs`
If an [ipfs URI](https://docs.ipfs.io/how-to/address-ipfs-on-web/#native-urls) is provided, it MUST resolve to an avatar image directly. Clients without built-in IPFS support MAY rewrite the URI to an https URL referencing an IPFS gateway as described in [this document](https://docs.ipfs.io/how-to/address-ipfs-on-web/) before resolving it as an https URL.

### `data`
If a [data URL](https://datatracker.ietf.org/doc/html/rfc2397) is provided, it MUST resolve to an avatar image directly.

### `ar`
If an [arweave URI](https://www.arweave.org/) is provided (ie. `ar://<transaction ID>`), it MUST resolve to an avatar image transaction directly. arweave files can be resolved easily using gateways such as [arweave.net](https://arweave.net), like so: https://arweave.net/DbzibnHnISvt-KbI6wmNfPeUmV89cEmDifsTgfM2DnA .

#### Mutable Transactions
Clients SHOULD support mutable update records on arweave by performing a lookup for a new "update" to a transaction using the arweave GraphQL API. All arweave gateways support the GraphQL API, for example the arweave.net endpoint is https://arweave.net/graphql .

Client can find the latest transaction for a transaction with ID `TRX_A` by searching for the latest record with the tag `Origin: TRX_A` with a transaction owner equal to the original transaction's owner. Clients can lookup the original transaction data using a gateway as so: https://arweave.net/tx/DbzibnHnISvt-KbI6wmNfPeUmV89cEmDifsTgfM2DnA . If no record exists with that tag, the original transaction is the "latest" record.

An example of this query for transaction ID `TRX_A` looks like this:

```graphql
{
  transactions(owners: ["<TRX_A owner address>"], tags: { name: "Origin", values: ["TRX_A"] }, sort: HEIGHT_DESC) {
    edges {
      node {
        id
      }
    }
  }
}
```

### NFTs
A reference to an NFT may be used as an avatar URI, following the standards defined in [CAIP-22](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-22.md) and [CAIP-29](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/CAIP-29.md).

Clients MUST support at least ERC721 and ERC1155 type NFTs, and MAY support additional types of NFT.

To resolve an NFT URI, a client follows this process:
 1. Retrieve the metadata URI for the token specified in the `avatar` field URI.
 2. Resolve the metadata URI, fetching the ERC721 or ERC1155 metadata.
 3. Extract the image URL specified in the NFT metadata.
 4. Resolve the image URL and use it as the avatar.

Clients MUST support at least `https` and `ipfs` URIs for resolving the metadata URI and the avatar image, and MAY support additional schemes. Clients MAY implement `ifps` scheme support by rewriting the URI to an HTTPS URL referencing an IPFS gateway as described above.

Clients SHOULD additionally take the following verification steps:
 1. Where the avatar URI was retrieved via forward resolution (starting from an ENS name), call the `addr` function on the same resolver and for the same name to retrieve the Ethereum address to which the name resolves. Otherwise, if the avatar URI was retrieved via reverse resolution (starting from an Ethereum address), use that address.
 2. Verify that the address from step 1 is an owner of the NFT specified in the URI. If it is not, the client MUST treat the URI as invalid and behave in the same manner as they would if no avatar URI was specified.

Clients MAY support NFT URIs by rewriting them to `https` URIs for a service that provides NFT avatar image resolution support.

## Examples

The following examples all resolve to the same avatar image:
```
eip155:1/erc721:0xbc4ca0eda7647a8ab7c2061c2e118a18a936f13d/0 # BAYC token 0
ipfs://QmRRPWG96cmgTn2qSzjwr2qvfNEuhunv6FNeMFGa9bx6mQ # IPFS hash for BAYC token 0 image
https://ipfs.io/ipfs/QmRRPWG96cmgTn2qSzjwr2qvfNEuhunv6FNeMFGa9bx6mQ # HTTPS URL to IPFS gateway for BAYC token 0 image
```
