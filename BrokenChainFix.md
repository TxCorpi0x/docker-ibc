# Broken Chain Fix

## Failure reason

Looking into the broken blockchain code, using a `map` iteration to put the loop data to be processed in the next loop to store the values to the blockchain state leads to a non-deterministic behavior and mentioned [here](https://docs.tendermint.com/v0.34/introduction/what-is-tendermint.html#a-note-on-determinism). Using random numbers, multithread processes such as goroutine, floating points and time (system time without considering the block time) which may cause different ordering in the final data. (Here)[https://ashourics.medium.com/the-challenge-of-gos-map-iteration-in-the-cosmos-sdk-blockchain-a-dive-into-determinism-bd5a99260519] is the complete explanation of `map` non-deterministic behavior. the culprit line of the code is:

```go
	for i := 0; i < types.NumIterations; i++ {
		mymap[i] = i
	}
```

The loop is adding the number to the map in the ascending order, but the map behind the scenes does not respect the input order and it is not sorted. as golang document insisted:

> When iterating over a map with a range loop, the iteration order is not specified and is not guaranteed to be the same from one iteration to the next. 

### Suggested Change

Remove the map iteration and use the store logic inside a for loop which has the same ordering in different processes and machines.
```go
func (k msgServer) Brkchain(goCtx context.Context, msg *types.MsgBrkchain) (*types.MsgBrkchainResponse, error) {
	ctx := sdk.UnwrapSDKContext(goCtx)

	for i := 0; i < types.NumIterations; i++ {

		store := prefix.NewStore(ctx.KVStore(k.storeKey), []byte(MyStoreKey))

		key := []byte(strconv.Itoa(i))
		if store.Has(key) {
			return nil, fmt.Errorf("key already exists in store")
		}

		value := []byte(strconv.Itoa(i))
		if len(value) == 0 {
			return nil, fmt.Errorf("value cannot be 0 length")
		}

		store.Set(key, value)
	}

	return &types.MsgBrkchainResponse{}, nil
}
```

to broadcast the transaction, it might be needed to append `--gas=auto` to the transaction command:

```bash
dosomething tx brkcosmossdk brkchain --from validator2 --home /tmp/val2 --keyring-backend test --fees 100stake --gas auto --chain-id dosomething -y
```
