# MCP Named Authorization Server Issuer

## Problem

LiteLLM's named MCP protected-resource metadata advertises an authorization server such as `https://gateway.example.com/slack`, but the corresponding authorization-server metadata reports `https://gateway.example.com` as its issuer. RFC 8414 section 3.3 requires the returned issuer to exactly match the authorization-server identifier used to derive the metadata URL, so strict MCP clients reject the metadata.

## Design

For named MCP authorization-server metadata, set `issuer` to `request_base_url/{mcp_server_name}`. Keep unnamed root discovery unchanged at `request_base_url`.

This applies to both named discovery route shapes because both call `_build_oauth_authorization_server_response` with the server name. Authorization, token, and registration endpoint URLs remain unchanged.

## Compatibility

Root `/.well-known/oauth-authorization-server` metadata retains its existing origin issuer. Named metadata becomes consistent with the existing path-scoped `authorization_servers` value and RFC 8414 path-based issuer discovery.

## Testing

Add regression coverage that verifies:

- named authorization-server metadata returns the path-scoped issuer
- DCR bridge metadata returns the same path-scoped issuer it advertises
- root authorization-server metadata retains the origin issuer
