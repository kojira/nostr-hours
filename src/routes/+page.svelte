<script lang="ts">
	import { onMount } from 'svelte';
	import { NostrFetcher } from 'nostr-fetch';
	import { nip19 } from 'nostr-tools';
	import type { Content, Event, Nip07 } from 'nostr-typedef';
	import { browser } from '$app/environment';
	import { page } from '$app/stores';

	let npub = $state('');
	let metadata: Content.Metadata | undefined = $state();
	let events: Event[] = $state.raw([]);

	const defaultRelays = ['wss://relay.nostr.band/', 'wss://nos.lol/'];
	const days = 14;
	let displayEventCount = $state(false);
	let displayGradation = $state(false);

	const now = new Date();
	const dates = Array.from({ length: days }, (_, i) => {
		const date = new Date(now.toDateString());
		date.setDate(now.getDate() - i);
		return date;
	});
	const hours = Array.from({ length: 24 }, (_, i) => i);

	onMount(() => {
		npub = $page.url.searchParams.get('npub') ?? '';
	});

	async function fetch(pubkey: string): Promise<void> {
		const { waitNostr } = await import('nip07-awaiter');
		const nostr = await waitNostr(1000);
		let relaysResultNIP07;
		try {
			relaysResultNIP07 = await nostr?.getRelays?.();
		} catch (error) {
			console.error(error);
		}
		const relaysWithNIP07 =
			relaysResultNIP07 !== undefined
				? [
						...new Set([
							...Object.entries(relaysResultNIP07).map(([relay]) => relay),
							...defaultRelays
						])
					]
				: defaultRelays;

		const relaysResultKind10002 = await getRelaysWithKind10002(relaysWithNIP07, pubkey);
		const relays =
			relaysResultKind10002 !== undefined
				? [
						...new Set([
							...Object.entries(relaysResultKind10002).map(([relay]) => relay),
							...relaysWithNIP07
						])
					]
				: relaysWithNIP07;

		const fetcher = NostrFetcher.init();
		fetcher.fetchLastEvent(relays, { kinds: [0], authors: [pubkey] }).then((event) => {
			if (event === undefined) {
				return;
			}
			try {
				metadata = JSON.parse(event.content);
			} catch (error) {
				console.warn('[failed to parse metadata]', error, event);
			}
		});
		const iterator = fetcher.allEventsIterator(
			relays,
			{ authors: [pubkey] },
			{
				since: Math.floor(dates[dates.length - 1].getTime() / 1000),
				until: Math.floor(dates[0].getTime() / 1000) + 1 * 24 * 60 * 60
			},
			{
				skipVerification: true
			}
		);
		for await (const event of iterator) {
			console.log(event);
			events = [...events, event];
		}
	}

	async function getRelaysWithKind10002(
		relays: string[],
		pubkey: string
	): Promise<Nip07.GetRelayResult | undefined> {
		const fetcher = NostrFetcher.init();
		const ev: Event | undefined = await fetcher.fetchLastEvent(relays, {
			kinds: [10002],
			authors: [pubkey]
		});
		if (ev === undefined) {
			return undefined;
		}
		const newRelays: Nip07.GetRelayResult = {};
		for (const tag of ev.tags.filter((tag) => tag.length >= 2 && tag[0] === 'r')) {
			newRelays[tag[1]] = {
				read: tag.length === 2 || tag[2] === 'read',
				write: tag.length === 2 || tag[2] === 'write'
			};
		}
		fetcher.shutdown();
		return newRelays;
	}

	async function inputNpub(): Promise<void> {
		const { waitNostr } = await import('nip07-awaiter');
		const nostr = await waitNostr(1000);
		if (nostr === undefined) {
			alert('Install NIP-07 browser extension');
			return;
		}
		const pubkey = await nostr.getPublicKey();
		npub = nip19.npubEncode(pubkey);
	}

	function calculateColorIntensity(eventsCount: number) {
		const maxEvents = 60;
		if (eventsCount === 0) {
			return 'rgb(255, 255, 255)';
		}
		if (!displayGradation) {
			return 'rgb(128, 0, 128)';
		}
		const red = 255 - (Math.min(eventsCount, maxEvents) / maxEvents) * 127;
		const green = 255 - (Math.min(eventsCount, maxEvents) / maxEvents) * 255;
		const blue = 255 - (Math.min(eventsCount, maxEvents) / maxEvents) * 127;

		return `rgb(${red}, ${green}, ${blue})`;
	}

	function totalEventsForDate(index: number) {
		return eventsCountPerHour[index] ? eventsCountPerHour[index].reduce((a, b) => a + b, 0) : 0;
	}

	$effect(() => {
		if (npub.startsWith('npub1') && browser) {
			console.log(npub);
			try {
				const { type, data: pubkey } = nip19.decode(npub);
				if (type === 'npub') {
					metadata = undefined;
					events = [];
					fetch(pubkey);
					history.replaceState(history.state, '', `${$page.url.pathname}?npub=${npub}`);
				}
			} catch (error) {}
		}
	});

	let eventsCountPerHour = $derived(
		dates.map((date) =>
			hours.map(
				(hour) =>
					events.filter((event) => {
						const createdAt = event.created_at * 1000;
						return (
							date.getTime() + hour * 60 * 60 * 1000 <= createdAt &&
							createdAt < date.getTime() + (hour + 1) * 60 * 60 * 1000
						);
					}).length
			)
		)
	);
</script>

<h1>Nostr hours</h1>
<p>How many hours do you spend in Nostr?</p>

<form onsubmit={(e) => e.preventDefault()}>
	<div>
		<input type="text" bind:value={npub} placeholder="npub1..." />
		<input type="button" onclick={inputNpub} value="from NIP-07" />
	</div>

	<div>
		<label>
			<input type="checkbox" bind:checked={displayEventCount} />
			Display Event Count
		</label>

		<label>
			<input type="checkbox" bind:checked={displayGradation} />
			Display Gradation
		</label>
	</div>
</form>

{#if metadata !== undefined}
	<article>
		<img src={metadata.picture} alt="" />
		<div>{metadata.display_name ? metadata.display_name : metadata.name}</div>
	</article>
{/if}

<table>
	<thead>
		<tr>
			<th></th>
			{#each hours as hour}
				<th>{hour}</th>
			{/each}
			<th></th>
		</tr>
	</thead>
	<tbody>
		{#each dates as date, index}
			<tr>
				<td class="day-{date.getDay()}">{date.toLocaleDateString()}</td>
				{#each hours as hour, hourIndex}
					<td
						style:background-color={calculateColorIntensity(eventsCountPerHour[index][hourIndex])}
					>
						{displayEventCount ? eventsCountPerHour[index][hourIndex] : ''}
					</td>
				{/each}
				<td>{displayEventCount ? totalEventsForDate(index) : ''}</td>
			</tr>
		{/each}
	</tbody>
</table>

<style>
	* {
		text-align: center;
	}

	form div {
		margin: 1rem auto;
	}

	img {
		width: 80px;
		height: 80px;
		object-fit: cover;
		border-radius: 50%;
	}

	table {
		margin: 2rem auto;
	}

	th,
	td {
		width: 2rem;
		text-align: center;
	}
	.day-0 {
		background-color: rgba(200, 50, 50, 0.2);
	}
	.day-6 {
		background-color: rgba(50, 50, 200, 0.2);
	}
</style>
