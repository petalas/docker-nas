# Note
If you get the error `router uses a non-existent resolver` it could be due to wrong permissions on `acme.json`
To reset it: `rm acme.json && touch acme.json && chmod 600 acme.json`

Don't reset too often, issuing new certs is rate-limited (5 every 168 hours).