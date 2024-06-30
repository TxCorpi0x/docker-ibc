# Inter-Chain Governance Proposal

## Requirements

1. Two chains with IBC and inter-chain modules enabled.
2. A Relayer, would be [Go Relayer](https://github.com/cosmos/relayer) or [Hermes](https://github.com/informalsystems/hermes)
3. Community interaction and Voting for Governance proposals.

## Steps

### Run Chains

Start two chains, Chain1 (Saga) and Chain2 (Osmosis) with [integrated](https://ibc.cosmos.network/main/ibc/integration/) IBC and Inter-Chain modules. Both chains should have access to each other via GRPC and RPC endpoints.

#### Create IBC Clients

Create a client for the Osmosis Chain on the Saga chain.

```bash
sagad tx ibc client create [path/to/client_state.json] [path/to/consensus_state.json] [flags]
```

### Configure Relayer

Download and install Hermes, configure it according to the Chain1 and Chain2 Configurations. Sample config between a Saga chain node and Osmosis.

```yml
[global]
log_level = 'info'
[mode]
[mode.clients]
enabled = true
refresh = true
misbehaviour = true
[mode.connections]
enabled = true
[mode.channels]
enabled = true
[mode.packets]
enabled = true
clear_interval = 100
clear_on_start = true
tx_confirmation = true
[telemetry]
enabled = true
host = '127.0.0.1'
port = 3001
[rest]
enabled = true
host    = '127.0.0.1'
port    = 3000


[[chains]]
id = 'saga-chain'
rpc_addr = 'http://chain1:26657'
grpc_addr = 'http://chain1:9090'
websocket_addr = 'ws:/chain1:26657/websocket'
rpc_timeout = '15s'
account_prefix = 'saga'
key_name = 'sagakey'
store_prefix = 'ibc'
gas_price = { price = 0.01, denom = 'usaga' }
max_gas = 10000000
clock_drift = '5s'
trusting_period = '1days'
trust_threshold = { numerator = '1', denominator = '3' }

[[chains]]
id = 'osmosis-chain'
rpc_addr = 'http://chain2:26657'
grpc_addr = 'http://chain2:9090'
websocket_addr = 'ws:/chain2:26657/websocket'
rpc_timeout = '15s'
account_prefix = 'osmo'
key_name = 'osmokey'
store_prefix = 'ibc'
gas_price = { price = 0.01, denom = 'uosmo' }
max_gas = 10000000
clock_drift = '5s'
trusting_period = '1days'
trust_threshold = { numerator = '1', denominator = '3' }
```

#### Import Keys

To interact and broadcast transactions, we need to import accounts (keys with seeds) to each counterparty chain.

```bash
hermes keys add --mnemonic-file saga.json --chain saga-chain
hermes keys add --mnemonic-file osmo.json --chain osmosis-chain 
```

#### Create Channel

Use hermes to create a channel between Chain1 and Chain2 alongside a new client connection, sample command:

```bash
hermes create channel --a-chain saga-chain --b-chain osmosis-chain --a-port transfer --b-port transfer --new-client-connection 
```

#### Start Relayer and Test

Start the relayer service
```bash
hermes start
```
and broadcast a sample ibc-transfer to make sure the channel is working fine.
```bash 
sagad tx ibc-transfer transfer transfer channel-0 saga1z9dhvw8ag3mrhyqr2kznxf4mv0nqamgz0qtnec 100000usaga --chain-id saga-chain --from sagakey --keyring-backend test --node tcp://chain1:26657 --broadcast-mode block --fees 10000000000000000usaga
```

### Inter-Chain accounts and Host and Controller chain

Saga chain as (controller chain), creates an inter-chain account on Osmosis chain (host chain). We are going to send transactions from saga chain to Osmosis to be broadcasted on the host chain.

```bash
sagad tx intertx register --from sagakey --connection-id connection-0 --chain-id saga-chain --home ./data/saga-chain --node tcp://chain1:16657 --keyring-backend test -y
```

#### Whitelist Messages

We need to whitelist the messages that we want to allow to be received by Osmosis, So we need a governance proposal and include `MsgSoftwareUpgrade` in the list of whitelisted messages of Osmosis in the inter-chain accounts parameters.

The same thing needed for Saga chain as well, so we have to allow the message via a governance proposal.

## Software Upgrade Proposal over IBC

The following steps needed for a software upgrade proposal to be submitted to host chain:

1. The Governance proposal needs a minimum deposit to be done by Saga chain. so Saga chain inter-chain account in Osmosis should have enough balance for to be used for deposit to the governance proposal. So minimum amount plus the transaction fees should be transferred and swapped in Osmosis.
2. Saga (controller) Broadcast Send-TX transaction over IBC channel to Osmosis(host).
3. For a software upgrade proposal needs votes, so the Osmosis Community should vote for `Yes` for the proposal to gets passed.

Sample SendTX code:

```go
func (k msgServer) SendTx(goCtx context.Context, msg *types.MsgSubmitTx) (*types.MsgSubmitTxResponse, error) {
	ctx := sdk.UnwrapSDKContext(goCtx)

	portID, err := icatypes.NewControllerPortID(msg.Owner)
	if err != nil {
		return nil, err
	}

	channelID, found := k.icaControllerKeeper.GetActiveChannelID(ctx, msg.ConnectionId, portID)
	if !found {
		return nil, sdkerrors.Wrapf(icatypes.ErrActiveChannelNotFound, "failed to retrieve active channel for port %s", portID)
	}

	chanCap, found := k.scopedKeeper.GetCapability(ctx, host.ChannelCapabilityPath(portID, channelID))
	if !found {
		return nil, sdkerrors.Wrap(channeltypes.ErrChannelCapabilityNotFound, "module does not own channel capability")
	}

	data, err := icatypes.SerializeCosmosTx(k.cdc, []sdk.Msg{msg.GetTxMsg()})
	if err != nil {
		return nil, err
	}

	packetData := icatypes.InterchainAccountPacketData{
		Type: icatypes.EXECUTE_TX,
		Data: data,
	}

	// timeoutTimestamp set to max value with the unsigned bit shifted to sastisfy hermes timestamp conversion
	// it is the responsibility of the auth module developer to ensure an appropriate timeout timestamp
	timeoutTimestamp := ^uint64(0) >> 1
	_, err = k.icaControllerKeeper.SendTx(ctx, chanCap, msg.ConnectionId, portID, packetData, timeoutTimestamp)
	if err != nil {
		return nil, err
	}

	return &types.MsgSubmitTxResponse{}, nil
}
```