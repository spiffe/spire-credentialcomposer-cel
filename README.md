# SPIRE Credential Composer CEL

[![Apache 2.0 License](https://img.shields.io/github/license/spiffe/helm-charts)](https://opensource.org/licenses/Apache-2.0)
[![Development Phase](https://github.com/spiffe/spiffe/blob/main/.img/maturity/dev.svg)](https://github.com/spiffe/spiffe/blob/main/MATURITY.md#development)

This project enables SPIRE Credential Composers to be written in [CEL](https://cel.dev/)

## Warning

This code is very early in development and is very experimental. Please do not use it in production yet. Please do consider testing it out, provide feedback, and maybe provide fixes.

## JWT Expressions

### Environment

The following root level variables are defined:
 * request - spire.plugin.server.credentialcomposer.v1.ComposeWorkloadJWTSVIDRequest
 * trust_domain - string, the trust domain of the server
 * spiffe_trust_domain - string, the trust domain in spiffe://<trust_domain> format

request has the following properties:
 * spiffe_id - string
 * attributes - spire.plugin.server.credentialcomposer.v1.JWTSVIDAttributes

request.attributes has the following properties:
 * claims - map(dyn, dyn)

### Macros

The standard macros are [available](https://github.com/google/cel-spec/blob/master/doc/langdef.md#macros).

Some ext macros are also availabe:
 * [cel.bind](https://pkg.go.dev/github.com/google/cel-go/ext#hdr-Cel_Bind-Bindings)
 * [strings](https://pkg.go.dev/github.com/google/cel-go/ext#Strings)
 * [two var comprehensions](https://pkg.go.dev/github.com/google/cel-go/ext#TwoVarComprehensions)

Custom macros are provided:
 * mapOverrideEntries - Runs on a map, give it another map and it will override settings in the first map with the second. It is a shallow override, no merging is performed.

### Return

Currently only the `spire.plugin.server.credentialcomposer.v1.ComposeWorkloadJWTSVIDResponse` type is
supported. It must be completely filled out. Other shortcut options may be added in the future.

## JWT Examples:

### Add `newkey=newvalue` to the token.

```
  CredentialComposer "cel" {
    plugin_cmd = "spire-credentialcomposer-cel"
    plugin_checksum = ""
    plugin_data {
      jwt {
        expression_string = <<EOB
spire.plugin.server.credentialcomposer.v1.ComposeWorkloadJWTSVIDResponse{
  attributes: spire.plugin.server.credentialcomposer.v1.JWTSVIDAttributes{
    claims: request.attributes.claims.mapOverrideEntries({
      'newkey': "newvalue"
    })
  }
}
EOB
      }
    }
  }
```

### Minio

In this example, we conditionally add a policy propery that is a list of properties as per the Minio OIDC 
documentation. The spiffe id path must start with /minio/ and everything after will be used as the policy
name.

For example, spiffe://example.org/minio/readonly will add to the token `policy: ["readonly"]`.

SPIRE Server Config:
```
  CredentialComposer "cel" {
    plugin_cmd = "spire-credentialcomposer-cel"
    plugin_checksum = ""
    plugin_data {
      jwt {
        expression_string = <<EOB
spire.plugin.server.credentialcomposer.v1.ComposeWorkloadJWTSVIDResponse{
  attributes: spire.plugin.server.credentialcomposer.v1.JWTSVIDAttributes{
    claims: request.attributes.claims.mapOverrideEntries(
      request.spiffe_id.startsWith(spiffe_trust_domain + "/minio/")?
      {'policy': [request.spiffe_id.substring(spiffe_trust_domain.size() + 7)]}:
      {}
    )
  }
}
EOB
      }
    }
  }
```

## CEL Hints

### Setting a variable:
```
cel.bind(varname, valueforvar,
  logic here
)
```

### Remove a specific item from a map
```
X.transformMap(k, v, k != 'abc', v)
```

### Update an existing item in a map:
```
X.transformMap(k, v, k == 'abc'? 72: v)
```

