<script lang="ts">
	import { ZipReader } from '@zip.js/zip.js';

	const LINE_REGEX = /(\r|\n)+/;
	const MESSAGE_LINK_REGEX = /https:\/\/discord.com\/channels\/[^\/]+\/(\d+)\//;

	let zipFileInput: HTMLInputElement;
	let duration = $state(-1);
	let channelId = $state('');
	let percentage = $state(0);
	let working = $state(false);

	const onprogress = async (progress: number, total: number) => {
		percentage = progress / total;
	};

	const checkForLinks = () => {
		const execResult = MESSAGE_LINK_REGEX.exec(channelId);
		if (execResult && execResult.length) {
			channelId = execResult[1];
		}
	};

	async function readDiscordLines(stream: ReadableStream<Uint8Array>) {
		const reader = stream.pipeThrough(new TextDecoderStream()).getReader();

		let lastLine = '';
		while (true) {
			const { done, value } = await reader.read();

			if (!value) {
				return;
			}

			const lines = (lastLine + value).split(LINE_REGEX);

			lines.forEach((line, index) => {
				if (index === lines.length - 1 && !done) {
					// parse this one on the next run
					lastLine = line;
					return;
				}

				if (line.includes('voice_disconnect') && (channelId === '' || line.includes(channelId))) {
					const parsedLine = JSON.parse(line);
					if (
						parsedLine['event_type'] === 'voice_disconnect' &&
						(channelId === '' || parsedLine['channel_id'] === channelId)
					) {
						duration += parseInt(parsedLine['duration']) || 0;
					}
				}
			});

			if (done) return;
		}
	}

	const zipSelected: EventListener = async (e) => {
		working = true;
        duration = 0;

		const file = zipFileInput.files?.[0];
		if (!file) return;

		const zipReader = new ZipReader(file.stream());

		const entries = await zipReader.getEntries();

		for (const eventsFile of entries.filter((entry) =>
			entry.filename.startsWith('activity/reporting/events')
		)) {
			const thisFileStream = new TransformStream();

			if (!eventsFile.getData) continue;

			await Promise.all([
				eventsFile.getData<ArrayBuffer>(thisFileStream.writable, { onprogress }),
				thisFileStream.readable,
				readDiscordLines(thisFileStream.readable)
			]);
		}

		zipReader.close();

		working = false;
	};
</script>

<svelte:head>
	<title>How many hours have you spent in Discord calls?</title>
</svelte:head>

<header>
	<h1>
		{duration === -1
			? 'How many hours have you spent in Discord calls?'
			: `${duration / 3_600_000} hours in calls`}
	</h1>
	<div>
		{#if channelId && duration !== -1}(in channel {channelId}){:else}Unofficial, not affiliated with
			Discord!{/if}
	</div>
</header>

<main>
	<div class:working>
		<div class="form-row">
			<label for="channelId"
				>Channel ID or link to message from channel (leave blank for total)</label
			>
			<input type="text" name="channelId" bind:value={channelId} onchange={checkForLinks} />
		</div>
		<div class="form-row">
			<label for="zipFile">package.zip file from Discord</label>
			<input type="file" name="zipFile" multiple={false} bind:this={zipFileInput} />
		</div>
		<button onclick={zipSelected}>Calculate!</button>
	</div>

	<progress value={percentage}>{percentage * 100}%</progress>

	<div class="explainer">
		<h2>How do I get a package.zip file from Discord?</h2>
		<p>
			Request a copy of your data from the Discord privacy settings page. <a
				href="https://support.discord.com/hc/en-us/articles/360004027692-Requesting-a-Copy-of-your-Data"
				>There's a how to guide in the help centre here.</a
			>
		</p>
		<h2>Are you stealing my messages?!</h2>
		<p>
			No, and I couldn't if I wanted to, as this is calculated entirely locally - your data doesn't
			ever leave your computer.
		</p>
		<p>
			You can <a href="https://github.com/mashedkeyboard/discord-talk-time">read the source code</a> to see what this is doing - or you can run it offline without
			your computer connected to the Internet, just save this page first!
		</p>
		<p>
			Discord data export files may be quite large, so you will probably want to run this on a
			computer, not a phone.
		</p>
		<h2>What's a channel ID?</h2>
		<p>
			If you want to get the amount of time you've spent in voice calls with a specific person or
			group, you can use the channel ID.
		</p>
		<h3>For DMs</h3>
		<p>
			To get this for a DM or group chat, right click any message in the DM, and click "Copy Message
			Link".
		</p>
		<p>
			You should get a link like <code>https://discord.com/channels/@me/number1/number2</code> -
			<code>number1</code> is the channel ID.
		</p>
		<p>
			If you're not sure, you can just paste the whole message link into the channel ID field and
			the website will sort it for you.
		</p>
		<h3>For voice channels</h3>
		<p>
			<a
				href="https://www.howtogeek.com/714348/how-to-enable-or-disable-developer-mode-on-discord/"
				rel="nofollow">Enable Discord developer mode</a
			>, then right-click the voice channel, and select "Copy Channel ID".
		</p>
	</div>
</main>

<footer>
	<p>
		Made with love in York by <a href="https://cpf.sh/">Curtis Parfitt-Ford</a>
	</p>
</footer>

<style lang="scss">
	$primary_light: #6a5acd;
	$primary_dark: #191970;

	header {
		width: 100%;
		padding: 1em;
		background-color: $primary_light;
		color: #f2f2f2;
	}

	main {
		width: 100%;
		height: 100%;
		max-width: 100vw;
		padding: 1em;
		padding-top: 0;
	}

	h2,
	h3 {
		color: $primary_dark;
	}

	footer {
		text-align: center;
	}

	progress {
		display: none;
		width: calc(100% - 2em);
		border: 1px solid grey;
		margin-top: 2em;
		accent-color: $primary_light;
	}

	.explainer {
		margin-right: auto;
		margin-left: auto;
	}

	.working {
		& ~ progress {
			display: block;
		}

		display: none;
	}

	.form-row {
		padding: 1em 0;
		> * {
			display: block;
			margin-top: 0.5em;
		}
	}

	button {
		padding: 1em;
	}
</style>
