<script>
	import * as openpgp from "openpgp";
	import { saveAs } from "file-saver";
	let privateKey = null;
	let privateKeyPassword = null;
	let token = "";
	let tokenUrl = "";
	let publicKey = null;
	let verificationResult = "";
	function getOwnerPublicKey(token) {
		// Owner's public key always located in the bottom of the list.
		const regexp =
			/-----BEGIN PGP PUBLIC KEY BLOCK-----.*?-----END PGP PUBLIC KEY BLOCK-----/gs;
		let keys = [...token.matchAll(regexp)].at(-1);
		if (keys.length > 0) {
			return keys.at(-1);
		}
		return "";
	}
	function readFileContent(f) {
		return f[0].text();
	}
	async function handleClick() {
		let ownerPrivateKey;
		try {
			ownerPrivateKey = await openpgp.decryptKey({
				privateKey: await openpgp.readKey({
					armoredKey: await readFileContent(privateKey),
				}),
				passphrase: privateKeyPassword,
			});
		} catch (error) {
			alert(
				"Failed to read your private key, make sure you select your private key not public key."
			);
			return;
		}
		let receiverPublicKey;
		try {
			receiverPublicKey = await readFileContent(publicKey);
		} catch (error) {
			alert(
				"Failed to read receiver's public key, make sure the public key's formate is correct."
			);
			return;
		}

		let tokenContent;
		if (token) {
			try {
				tokenContent = await readFileContent(token);
			} catch (error) {
				alert("Failed to read token content.");
			}
		} else {
			tokenContent = `Initial token for ${tokenUrl}\n`;
		}

		let signedMessage;
		try {
			const message = await openpgp.createCleartextMessage({
				text: `${tokenContent} \nI transfer token ownership to another person. His/Her public key is: \n${receiverPublicKey}\n`,
			});
			signedMessage = await openpgp.sign({
				message: message,
				signingKeys: ownerPrivateKey,
			});
		} catch (error) {
			alert("Failed to mint token.");
		}

		try {
			await verify(signedMessage);
		} catch (error) {
			alert(
				"Token verification failed. Make sure you are the owner of the token and use correct private/public key." +
					error
			);
			return;
		}

		const blob = new Blob([signedMessage], {
			type: "text/plain;charset=utf-8",
		});
		saveAs(blob, "token.txt");
	}

	async function verify(tokenContent) {
		const PGP_MESSAGE_END = "-----END PGP SIGNATURE-----";
		while (true) {
			let ownerPublicKeyArmored = getOwnerPublicKey(
				(await openpgp.unarmor(tokenContent)).text
			);
			const publicKey = await openpgp.readKey({
				armoredKey: ownerPublicKeyArmored,
			});
			const verifyResult = await openpgp.verify({
				message: await openpgp.readCleartextMessage({
					cleartextMessage: tokenContent,
				}),
				verificationKeys: publicKey,
			});
			await verifyResult.signatures[0].verified;
			tokenContent = verifyResult.data;
			const nestedMessageIndex =
				tokenContent.lastIndexOf(PGP_MESSAGE_END);
			if (nestedMessageIndex == -1) {
				break;
			}
			tokenContent = tokenContent.substring(
				0,
				nestedMessageIndex + PGP_MESSAGE_END.length
			);
		}
		return true;
	}
	async function verifyToken() {
		let tokenContent = await readFileContent(token);
		try {
			await verify(tokenContent);
		} catch (error) {
			verificationResult = error;
			return;
		}
		verificationResult = "success";
	}
</script>

<main>
	<h1>A “NFT” without blockchain</h1>
	<p>
		I found the article <a
			href="https://shkspr.mobi/blog/2021/12/an-nft-without-a-blockchain-no-gas-fees-no-eth/"
			>“An NFT without a Blockchain”</a
		> quite interesting and decided to implement it for demo purposes.
	</p>
	<h2>Create or Transfer your ChainlessNFT</h2>
	<p>
		You need to have your private key and receiver's public key before using
		this tool. You can create one using <a href="https://pgpkeygen.com"
			>pgpkeygen.com</a
		>
	</p>
	<div>
		<h3>
			1. Select the generated PGP private file and enter the password for
			the key.
		</h3>
		<label for="privatekey">Select your private key: </label>
		<input type="file" id="privatekey" bind:files={privateKey} />
		<label for="password">Your private key password: </label>
		<input
			type="password"
			id="privatekeypassword"
			bind:value={privateKeyPassword}
		/>
	</div>
	<div>
		<h3>
			2. Select your existing ChainlessNFT file or create a new one from a
			link.
		</h3>
		<label for="token">Select your token file: </label>
		<input type="file" id="token" bind:files={token} />
		<label for="url">Or use a URL to create your own token.</label>
		<input type="text" id="url" bind:value={tokenUrl} />
	</div>
	<div>
		<h3>3. Upload the public PGP key of the receiver</h3>
		<label for="publickey">Select receiver's public key: </label>
		<input type="file" id="publickey" bind:files={publicKey} />
	</div>
	<div>
		<button on:click={handleClick}>Mint it!</button>
	</div>
	<h2>Verify ChainlessNFT Token</h2>
	<div>
		<label for="token_verify"
			>Select your token file for verification:
		</label>
		<input type="file" id="token_verify" bind:files={token} />
		<button on:click={verifyToken}>Verify it!</button>
		{#if verificationResult == ""}
			<div />
		{:else if verificationResult == "success"}
			<p>Token is valid!</p>
		{:else}
			<p>Token is invalid! {verificationResult}</p>
		{/if}
	</div>
	<h2>Q & A</h2>
	<ol>
		<li class="has-line-data" data-line-start="0" data-line-end="2">
			<b>Is it secure?</b> <br />
			Yes, all the computation is done locally. It will not upload any data
			to any server.
		</li>
		<li class="has-line-data" data-line-start="2" data-line-end="4">
			<b>How much does it cost?</b><br />
			Free
		</li>
		<li class="has-line-data" data-line-start="4" data-line-end="6">
			<b>How can I store the ChainlessNFT?</b><br />
			Save it in your local disk or in cloud storage services like Google Drive
			or Dropbox.
		</li>
		<li class="has-line-data" data-line-start="6" data-line-end="8">
			<b>What if I lost the ChainlessNFT?</b><br />
			There is no recovery mechanism, the same thing happens to your NFTs if
			you lose your private key.
		</li>
		<li class="has-line-data" data-line-start="8" data-line-end="10">
			<b>Can you mint your token to different people?</b><br />
			Yes, you can. But you can do that with every NFT as well. There is nothing
			preventing you from creating your own NFT contract(ERC721) and claiming
			your ownership of a hash.
		</li>
		<li class="has-line-data" data-line-start="10" data-line-end="12">
			<b>How can I verify the time of transaction?</b><br />
			You can use services like free TSA (<a
				href="https://www.freetsa.org/index_en.php"
				>https://www.freetsa.org/index_en.php</a
			>) to create a timestamp of your ChainlessNFT.
		</li>
		<li class="has-line-data" data-line-start="12" data-line-end="14">
			<b>Why should I trust the free TSA?</b><br />
			NFTers still have to trust NFT websites. NFT usually stores a URL of
			an image hosted by some websites. They are free to remove/update the image
			afterwards.
		</li>
	</ol>
</main>

<style>
	main {
		text-align: center;
		padding: 1em;
		max-width: 240px;
		margin: 0 auto;
	}

	h1 {
		color: #ff3e00;
		text-transform: uppercase;
		font-size: 4em;
		font-weight: 100;
	}

	@media (min-width: 640px) {
		main {
			max-width: none;
		}
	}

	li {
		list-style-position: inside;
	}
</style>
