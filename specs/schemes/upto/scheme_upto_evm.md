# Scheme: `upto` `EVM`

## Summary

The `upto` scheme on EVM chains uses `EIP-2612` and the `Permit2` contract to authorize a transfer of up to a specified amount of an `ERC20 token` from the payer to the resource server. This approach allows the resource server to charge the client for the actual cost of the resource, which may be less than or equal to the maximum amount specified.

## `X-Payment` header payload

The `payload` field of the `X-PAYMENT` header must contain the following fields:
- `signature`: The signatures for the `EIP-2612` `permit` operation and `Permit2` `permitTransferFrom` operation.
- `authorization`: parameters required to reconstruct the message signed for the permitted operations.

Example:

```
{
  "signature": {
    "permit": "0x2d6a7588d6acca505cbf0d9a4a227e0c52c6c34008c8e8986a1283259764173608a2ce6496642e377d6da8dbbf5836e9bd15092f9ecab05ded3d6293af148b571c",
    "transfer": "0x2d6a7588d6acca505cbf0d9a4a227e0c52c6c34008c8e8986a1283259764173608a2ce6496642e377d6da8dbbf5836e9bd15092f9ecab05ded3d6293af148b571c"
  }
  "authorization": {
    "permit": {
      "owner": "0x857b06519E91e3A54538791bDbb0E22373e36b66",
      "spender": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
      "value": "10000",
      "deadline": "1740672154",
      "nonce": "0xf3746613c2d920b5fdabc0856f2aeb2d4f88ee6037b8cc5d04a71a4462f13480"
    },
    "transfer": {
      "permitted": {
        "to": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
        "requestedAmount": "10000",
      },
      "nonce": "0xf3746613c2d920b5fdabc0856f2aeb2d4f88ee6037b8cc5d04a71a4462f13480",
      "deadline": "1740672154"
    }
  }
}

```

Full `X-PAYMENT` header:

```
{
  x402Version: 1,
  scheme: "upto",
  network: "base-sepolia",
  payload: {
    "signature": {
      "permit": "0x2d6a7588d6acca505cbf0d9a4a227e0c52c6c34008c8e8986a1283259764173608a2ce6496642e377d6da8dbbf5836e9bd15092f9ecab05ded3d6293af148b571c",
      "transfer": "0x2d6a7588d6acca505cbf0d9a4a227e0c52c6c34008c8e8986a1283259764173608a2ce6496642e377d6da8dbbf5836e9bd15092f9ecab05ded3d6293af148b571c"
    },
    "authorization": {
      "permit": {
        "owner": "0x857b06519E91e3A54538791bDbb0E22373e36b66",
        "spender": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
        "value": "10000",
        "deadline": "1740672154",
        "nonce": "0xf3746613c2d920b5fdabc0856f2aeb2d4f88ee6037b8cc5d04a71a4462f13480"
      },
      "transfer": {
        "permitted": {
          "to": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
          "requestedAmount": "10000",
        },
        "nonce": "0xf3746613c2d920b5fdabc0856f2aeb2d4f88ee6037b8cc5d04a71a4462f13480",
        "deadline": "1740672154"
      }
    }
  }
}
```

## Verification

Steps to verify a payment for the `upto` scheme:

1. Verify the signatures are valid
2. Verify the `client` has enough of the `asset` (ERC20 token) to cover `paymentRequirements.maxAmountRequired`
3. Verify the value in the `payload.authorization` is enough to cover `paymentRequirements.maxAmountRequired` (both permit and transfer)
4. Verify the spender in the `payload.authorization` is the `Permit2` contract
5. Varify the transfer `to` address is the resource server
6. Verify the transfer deadline is not expired
7. Verify nonce is not used
8. Verify the permit and transfer parameters are for the agreed upon ERC20 contract and chain
8. Simulate the `permit` and `permitTransferFrom` multicall to ensure the transaction would succeed

## Settlement

The resource server can settle the payment by first calling the `permit` function on the `EIP-2612` compliant contract giving allowance to use the tokens for the `Permit2` contract. Then, the resource server can call the `permitTransferFrom` function on the `Permit2` contract to transfer the tokens from the client to itself. The `Permit2` contract acts as a routing contract that's needed because the ERC20 `transferFrom` function has a hard dependency on the `msg.sender` so it would not be possible to call `permit` and `transferFrom` through a multicall.
With `Permit2`, both transactions become independent from the `msg.sender` and can be executed with multicall.

## Appendix

### Considerations
The `EIP-2612` `permit` uses incremental nonces which may cause nonce collisions or incorrect nonces (e.g., if a transaction is dropped or fails). This could be solved by requiring only one "infinite" approval if the user hasn't already approved the `Permit2` contract and handle all later transactions only through the `Permit2` contract. The `Permit2` contract uses random nonces so it would allow for multiple transactions to be executed in parallel without any issues. 