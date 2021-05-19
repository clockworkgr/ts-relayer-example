<template>
	<div class="example">
		<h2>This is a ts-relayer-demo</h2>
		<div class="logger">
			<template
				v-for="(log, index) in relayerLog"
				v-bind:key="'logline' + index"
			>
				<div>{{ log }}</div>
			</template>
		</div>
		<h3>
			Relayer Logger output above.<br />Please open dev-tools/console log for
			additional details.
		</h3>
		<div v-if="step == 1">
			<h4>Enter details for chain A</h4>
			<input
				v-model="config.chainA.endpoint"
				placeholder="RPC Endpoint"
			/><br />
			<input
				v-model="config.chainA.addrPrefix"
				placeholder="Address Prefix"
			/><br />
			<input
				v-model="config.chainA.gasPrice"
				placeholder="Gas Price (e.g. 0.001uatom)"
			/><br />
			<textarea v-model="config.chainA.mnemonic" placeholder="Mnemonic" /><br />
			<button v-on:click="next">Next</button>
		</div>
		<div v-if="step == 2">
			<h4>Enter details for chain B</h4>
			<input
				v-model="config.chainB.endpoint"
				placeholder="RPC Endpoint"
			/><br />
			<input
				v-model="config.chainB.addrPrefix"
				placeholder="Address Prefix"
			/><br />
			<input
				v-model="config.chainB.gasPrice"
				placeholder="Gas Price (e.g. 0.001uatom)"
			/><br />
			<textarea v-model="config.chainB.mnemonic" placeholder="Mnemonic" /><br />
			<button v-on:click="next">Next</button>
		</div>
		<div v-if="step == 3">
			<button v-on:click="setupRelayer">Connect</button>
		</div>
		<div v-if="step == 4">
			<button v-on:click="runRelayer">Run Relayer</button>
		</div>
		<div v-if="step == 5">
			<button v-on:click="stopRelayer">Stop Relayer</button><br />
			<input
				v-model="transaction.senderMnemonic"
				placeholder="Sender mnemonic on chain A"
			/><br />
			<input
				v-model="transaction.rcptAddress"
				placeholder="Recipient address on chain b"
			/><br />
			<input
				v-model="transaction.amount"
				placeholder="Amount to transfer (e.g. 1token)"
			/><br />
			<button v-on:click="sendIBCtransaction">
				Send IBC Transfer Transaction
			</button>
		</div>
	</div>
</template>

<script>
	import { stringToPath } from "@cosmjs/crypto";
	import { IbcClient, Link } from "@confio/relayer/build";
	import { DirectSecp256k1HdWallet } from "@cosmjs/proto-signing";
	import { GasPrice, parseCoins } from "@cosmjs/launchpad";
	import { SigningStargateClient } from "@cosmjs/stargate";
	import Long from "long";
	export default {
		name: "App",
		data() {
			return {
				step: 1,
				config: {
					chainA: {
						endpoint: "http://localhost:26657",
						addrPrefix: "mars",
						gasPrice: "0.0001token",
						mnemonic:
							"shrug eager kid scale grunt trust slice dose equal jacket speak book arctic spy time trouble output steel provide leave shuffle play wear expand",
					},
					chainB: {
						endpoint: "http://localhost:26659",
						addrPrefix: "planet",
						gasPrice: "0.0001token",
						mnemonic:
							"lock play cabin penalty trumpet permit correct dinner exchange usage focus normal net tongue present amused bid plunge hill dutch milk mobile obey diagram",
					},
				},
				transaction: {
					senderMnemonic: "",
					amount: "",
					rcptAddress: "",
				},
				signerA: null,
				signerB: null,
				clientA: null,
				clientB: null,
				link: null,
				channels: null,
				relayerLog: [],
				running: false,
				nextRelay: {},
			};
		},
		methods: {
			next() {
				this.step++;
			},
			prev() {
				this.step > 0 ? (this.step = this.step - 1) : (this.step = 0);
			},
			logger() {
				return {
					log: (msg) => {
						this.relayerLog.push("LOG: " + msg);
					},
					info: (msg) => {
						this.relayerLog.push("INFO: " + msg);
					},
					error: (msg) => {
						this.relayerLog.push("ERROR: " + msg);
					},
					warn: (msg) => {
						this.relayerLog.push("WARN: " + msg);
					},
					verbose: (msg) => {
						this.relayerLog.push("VERBOSE: " + msg);
					},
					debug: (msg) => {
						this.relayerLog.push("DEBUG: " + msg);
					},
				};
			},
			sleep(ms) {
				return new Promise((resolve) => setTimeout(resolve, ms));
			},
			async sendIBCtransaction() {
				const signer = await DirectSecp256k1HdWallet.fromMnemonic(
					this.transaction.senderMnemonic,
					{
						hdPaths: [stringToPath("m/44'/118'/0'/0/0")],
						addrPrefix: this.config.chainA.addrPrefix,
					}
				);
				const client = await SigningStargateClient.connectWithSigner(
					this.config.chainA.endpoint,
					signer
				);
				const [fromAccount] = await signer.getAccounts();
				const msg = {
					typeUrl: "/ibc.applications.transfer.v1.MsgTransfer",
					value: {
						sourcePort: "transfer",
						sourceChannel: this.channels.src.channelId,
						sender: fromAccount.address,
						receiver: this.transaction.rcptAddress,
						timeoutTimestamp: Long.fromString(
							new Date().getTime() + 60000 + "000000"
						),
						token: parseCoins(this.transaction.amount)[0],
					},
				};

				const fee = {
					amount: [],
					gas: "200000",
				};

				await client.signAndBroadcast(fromAccount.address, [msg], fee);
			},
			async relayerLoop(
				options = { poll: 5000, maxAgeDest: 86400, maxAgeSrc: 86400 }
			) {
				while (this.running) {
					try {
						this.nextRelay = await this.link.checkAndRelayPacketsAndAcks(
							this.nextRelay,
							2,
							6
						);
						console.group("Next Relay:");
						console.log(this.nextRelay);
						console.groupEnd("Next Relay:");
						await this.link.updateClientIfStale("A", options.maxAgeDest);
						await this.link.updateClientIfStale("B", options.maxAgeSrc);
					} catch (e) {
						console.error(`Caught error: `, e);
					}
					await this.sleep(options.poll);
				}
			},
			runRelayer() {
				this.running = true;
				this.relayerLoop();
				this.next();
			},

			stopRelayer() {
				this.running = false;
				this.prev();
			},
			async setupRelayer() {
				// Create a signer from chain A config
				this.signerA = await DirectSecp256k1HdWallet.fromMnemonic(
					this.config.chainA.mnemonic,
					{
						hdPaths: [stringToPath("m/44'/118'/0'/0/0")],
						addrPrefix: this.config.chainA.addrPrefix,
					}
				);
				// Create a signer from chain B config
				this.signerB = await DirectSecp256k1HdWallet.fromMnemonic(
					this.config.chainB.mnemonic,
					{
						hdPaths: [stringToPath("m/44'/118'/0'/0/0")],
						addrPrefix: this.config.chainB.addrPrefix,
					}
				);
				// get addresses
				const [accountA] = await this.signerA.getAccounts();
				const [accountB] = await this.signerB.getAccounts();
				// Create IBC Client for chain A
				this.clientA = await IbcClient.connectWithSigner(
					this.config.chainA.endpoint,
					this.signerA,
					accountA.address,
					{
						prefix: this.config.chainA.addrPrefix,
						logger: this.logger(),
						gasPrice: GasPrice.fromString(this.config.chainA.gasPrice),
					}
				);
				console.group("IBC Client for chain A");
				console.log(this.clientA);
				console.groupEnd("IBC Client for chain A");
				// Create IBC Client for chain B
				this.clientB = await IbcClient.connectWithSigner(
					this.config.chainB.endpoint,
					this.signerB,
					accountB.address,
					{
						prefix: this.config.chainB.addrPrefix,
						logger: this.logger(),
						gasPrice: GasPrice.fromString(this.config.chainB.gasPrice),
					}
				);
				console.group("IBC Client for chain B");
				console.log(this.clientB);
				console.groupEnd("IBC Client for chain B");

				// Create new connectiosn for the 2 clients
				this.link = await Link.createWithNewConnections(
					this.clientA,
					this.clientB,
					this.logger()
				);

				console.group("IBC Link Details");
				console.log(this.link);
				console.groupEnd("IBC Link Details");
				// Create a channel for the connections
				this.channels = await this.link.createChannel(
					"A",
					"transfer",
					"transfer",
					1,
					"ics20-1"
				);
				console.group("IBC Channel Details");
				console.log(this.channels);
				console.groupEnd("IBC Channel Details");
				this.next();
			},
		},
	};
</script>

<style>
	#app {
		font-family: Avenir, Helvetica, Arial, sans-serif;
		-webkit-font-smoothing: antialiased;
		-moz-osx-font-smoothing: grayscale;
		text-align: center;
		color: #2c3e50;
		margin-top: 60px;
	}

	.example {
		width: 720px;
		margin: auto;
	}
	.logger {
		width: 720px;
		overflow: auto;
		height: 400px;
		border: 1px solid black;
		text-align: left;
		padding: 5px;
	}
	input {
		width: 300px;
		margin: 10px auto;
		padding: 5px;
	}
	textarea {
		width: 300px;
		margin: 10px auto;
		height: 80px;
		padding: 5px;
	}
</style>
