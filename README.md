# opa-jwks-jwt

An example policy showing JSON Web Key Sets (JWKS) based validation of JSON Web Tokens (JWT) using the OPA [Token Verification](https://www.openpolicyagent.org/docs/latest/policy-reference/#token-verification) built-in functions .  

For simplicity, this example uses JWKS and JWT values generated with the sample JWKS service at: https://jwks-service.appspot.com/.  You can substitute your own values as appropriate for your environment.

## Setup
The organization of the policy rules and data will follow the OPA [Bundle File Format](https://www.openpolicyagent.org/docs/latest/management/#bundle-file-format)

### Get the JWKS 
Retrive the JKWS from the sample service and save as data in the OPA bundle hierarchy as `jwks/data.json`
```
curl https://jwks-service.appspot.com/.well-known/jwks.json -o bundle/jwks/data.json
```

### Generate a RS512 Token
You can generate a token via the web UI at https://jwks-service.appspot.com or via the following commands:
```
# As the service hosts multiple RSA keys, retrieve the first one:
export RSA_KEYID=$(curl https://jwks-service.appspot.com/keyids\?type\=rsa | jq -r '.ids[0]')
echo $RSA_KEYID
```

Create the payload for the token creation request
```
cat <<EOF > token_request_payload.json
{
  "keyid": "${RSA_KEYID}",
  "alg": "RS512",
  "expiry" : "300s",
  "notbefore" : "3s",
  "payloadclaims": {
    "aud": "opa-jwks-jwt-example",
    "iss": "jwks-service.appspot.com",
    "sub": "alice"
  }
}
EOF
```

Generate the token
```
curl https://jwks-service.appspot.com/token -X POST -H content-type:application/json -d @token_request_payload.json > token.txt
```

Create the `input.json` for the OPA eval query
```
cat <<EOF > input.json
{
  "jwt": "$(cat token.txt)",
}
EOF
```

## Test the Policy
```
# Verify the Signature only
opa eval -b ./bundle -i input.json 'data.rules.verify_output'
# result will contain `true`

# Decode the JWT only
opa eval -b ./bundle -i input.json 'data.rules.decode_output'
# result will contain the decoded token as JSON

# Decode AND Verify
opa eval -b ./bundle -i input.json 'data.rules.decode_verify_output'
# result will contain `true` AND the decoded token as JSON
```
