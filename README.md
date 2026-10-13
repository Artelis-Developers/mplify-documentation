# Cegecom MPLIFY API documentation

Client integration reference and examples for Cegecom's MEF LSO Sonata (Billie)
address validation, product qualification, quoting and ordering APIs.

- [Main API documentation](DOCUMENTATION.md) — production environment.
- [Staging API documentation](DOCUMENTATION-STAGING.md) — integration and testing.

## API environments

The documentation uses `https://<host>/<stage>/billie`. For both environments,
`host` is `zexbvfc9hj.execute-api.eu-central-1.amazonaws.com`.

- **Main:** `stage` is `main`.
  Base URL: `https://zexbvfc9hj.execute-api.eu-central-1.amazonaws.com/main/billie`
- **Staging:** `stage` is `staging`.
  Base URL: `https://zexbvfc9hj.execute-api.eu-central-1.amazonaws.com/staging/billie`

Use the reference for your chosen environment; available behaviour may differ.
Cegecom supplies your API key separately. Send its value in the `x-api-key` header;
do not put credentials in URLs or committed files.

Start with the worked integration examples in the appropriate reference. For access,
catalogue details or integration questions, contact your Cegecom representative.
